# opencode V1 上下文管理机制详解

> 范围：**仅 V1**，即 `packages/opencode/src/**` 与 `packages/core/src/v1/**`。
> 不涉及 V2（`packages/core/src/system-context`、Context Epoch、session projector、`/api/*` 服务面）。
> 所有结论均给出 `文件:行号`，可逐条核对。源码版本：opencode `1.18.34`。

---

## 0. 总览

V1 的上下文管理是**「每轮从数据库重新组装」**模型，没有内存里的"上下文对象"被长期持有：

```
外部 POST /session/:id/message
        │
        ▼
SessionPrompt.prompt() ──▶ 写 user message (+parts) ──▶ SessionPrompt.loop()
                                                              │
                                        ┌─────────────────────┴─────────────────────┐
                                        │  while(true)  ← 每个 step 循环一次         │
                                        │                                           │
                                        │  1. filterCompactedEffect(sessionID)      │ 全量历史读 DB
                                        │  2. latest(msgs) → lastUser 等            │
                                        │  3. agent = agents.get(lastUser.agent)     │ agent 来自消息
                                        │  4. SessionReminders.apply(...)           │ plan 等提醒注入
                                        │  5. 组装 system（6 段）                    │
                                        │  6. toModelMessagesEffect(msgs, model)     │ 历史 → 模型格式
                                        │  7. registry.tools(...) → 工具集           │ 按 agent/permission
                                        │  8. handle.process(...) → LLM 流           │
                                        │  9. 流事件落库为 part                      │
                                        │ 10. 判定 continue/compact/break            │
                                        └───────────────────────────────────────────┘
                                                              │
                                                              ▼
                                            返回最后一条 assistant message
```

三条主线：

| 维度 | 机制 | 一句话结论 |
|---|---|---|
| **System 提示词** | 6 段拼接，每轮现算 | agent 人格是第 1 段；AGENTS.md 每 step 重读 |
| **对话历史** | 每轮全量重读 + `filterCompacted` 重排 | **没有滑动窗口**，只有 prune + compaction 两道闸 |
| **Subagent** | 独立 `session` 行、独立历史、独立 system | 父会话内容**零共享**，只传 prompt 文本、只回最终文本 |

---

## 1. 数据模型：上下文存在哪

上下文全部来自三张表（同一 SQLite 库），没有第二份副本：

```ts
// packages/core/src/session/sql.ts:22-66  session
id, project_id, workspace_id, parent_id, slug, directory, path, title, version,
share_url, summary_*, metadata, cost, tokens_*, revert, permission, agent, model,
time_created/updated, time_compacting, time_archived

// :68-80  message（正文是 JSON）
id, session_id → session.id (cascade), time_*, data(json)

// :82-98  part（同样是 JSON）
id, message_id → message.id (cascade), session_id, time_*, data(json)
```

part 的类型决定它如何进入上下文（`packages/schema/src/v1/session.ts`）：

| part type | 是否进模型上下文 | 作用 |
|---|---|---|
| `text` | ✅ 原样 | 用户输入 / assistant 输出 |
| `reasoning` | ✅（同模型时带 providerMetadata；跨模型降级成 text） | 思考块 |
| `tool` | ✅（入参 + 结果，或 `[Old tool result content cleared]`） | 工具调用 |
| `step-start` / `step-finish` | ❌（只做结构/快照/统计） | 步骤边界、snapshot、token 记账 |
| `patch` | ❌ | 文件变更清单 |
| `compaction` | ✅ 转成 `"What did we do so far?"` | 压缩标记 + `tail_start_id` |
| `subtask` | ✅ 转成 `"The following tool was executed by the user"` | 命令驱动 subagent |
| `file` / `agent` | ✅（file 视 mime；`text/plain` 与目录不转发） | 附件 / `@agent` 提及 |

---

## 2. 一次 prompt 的完整时序

主循环：`packages/opencode/src/session/prompt.ts:1085-1339`。

```ts
while (true) {                                                   // :1088
  yield* status.set(sessionID, { type: "busy" })
  let msgs = yield* MessageV2.filterCompactedEffect(sessionID)    // :1092 ★ 每 step 重读全量历史
  const { user: lastUser, assistant: lastAssistant, finished, tasks } = MessageV2.latest(msgs)  // :1096

  if (!lastUser) throw new Error("No user message found in stream...")   // :1098

  // —— 溢出判定（入口处）——
  if (... isOverflow ...) {                                       // :1160-1168
    yield* compaction.create({ sessionID, agent: lastUser.agent, model: lastUser.model, auto: true })
    continue
  }

  const agent = yield* agents.get(lastUser.agent)                 // :1170 ★ agent 来自消息
  const maxSteps = agent.steps ?? Infinity                        // :1178
  const isLastStep = step >= maxSteps                             // :1179
  msgs = yield* SessionReminders.apply({ messages: msgs, agent, session })   // :1180

  // —— 写 assistant 消息骨架 ——
  const msg: SessionV1.Assistant = { id: MessageID.ascending(), parentID: lastUser.id,
    mode: agent.name, agent: agent.name, path: {cwd, root}, tokens: {...}, ... }   // :1186-1200
  yield* sessions.updateMessage(msg)                              // :1201

  // —— 组装上下文 ——
  yield* plugin.trigger("experimental.chat.messages.transform", {}, { messages: msgs })  // :1255
  const [skills, env, instructions, mcpInstructions, modelMsgs] = yield* Effect.all([...])  // :1257-1263
  const system = [...env, ...instructions, ...(mcp), ...(skills)]  // :1264-1269
  const result = yield* handle.process({
    user: lastUser, agent, permission: session.permission, sessionID,
    parentSessionID: session.parentID, system,
    messages: [...modelMsgs, ...(isLastStep ? [{ role: "assistant", content: MAX_STEPS_PROMPT }] : [])],
    tools, model, toolChoice,
  })                                                              // :1272-1286

  // —— 收尾判定 ——
  if (result === "stop") return "break"                           // :1319
  if (result === "compact") yield* compaction.create({ auto: true, overflow: !handle.message.finish })  // :1320-1328
  return "continue"
}

yield* compaction.prune({ sessionID }).pipe(Effect.ignore, Effect.forkIn(scope))   // :1338 循环结束后异步 prune
return yield* lastAssistant(sessionID)                            // :1339
```

关键结论：

1. **每 step 重新读库**（`:1092`）：工具结果、摘要、revert 在下一轮自动生效，无需维护内存上下文。
2. **agent 是"消息级"属性**（`:1170` + `:475` 的 `userMsg.agent`）：换 agent = 发一条新 user 消息，历史不重置。
3. **`latest()` 按 `time.created` 判断最新**（`message-v2.ts:604-608`），因为 `filterCompacted` 会**重排数组**。
4. `step` 计数在循环里自增，与 `agent.steps` 共同决定何时注入 `MAX_STEPS_PROMPT`（`:1281`，文本见 `packages/core/src/session/runner/max-steps.ts`）。
5. 命令驱动的 subagent 通过 `latest()` 返回的 `tasks` 进入（`:1096` → `:256-430`）。

---

## 3. System 提示词的 6 段拼装

两处拼接，顺序固定：

```ts
// session/prompt.ts:1257-1269  （①-④ 段）
const [skills, env, instructions, mcpInstructions, modelMsgs] = yield* Effect.all([
  sys.skills(agent), sys.environment(model),
  instruction.system(), sys.mcp(agent, session.permission),
  MessageV2.toModelMessagesEffect(msgs, model),
])
const system = [
  ...env,                                          // ①
  ...instructions,                                 // ②
  ...(mcpInstructions ? [mcpInstructions] : []),   // ③
  ...(skills ? [skills] : []),                     // ④
]
```

```ts
// session/llm/request.ts:58-66  （⑤⑥ 段 + 最终顺序）
const system = [[
  ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),  // ⑤
  ...input.system,                                                                     // ①-④
  ...(input.user.system ? [input.user.system] : []),                                   // ⑥
].filter((x) => x).join("\n")]      // ← 全部压成一个字符串
```

| 段 | 内容 | 产出函数 | 随 agent | 随轮次 | 随目录 |
|---|---|---|---|---|---|
| ⑤ | agent 人格：`agent.<name>.prompt`，否则按模型 id 选 `PROMPT_DEFAULT/GPT/ANTHROPIC/GEMINI/BEAST/ASTRA/CODEX/KIMI/META/TRINITY` | `agent/agent.ts:283`、`session/system.ts:28-51`、`llm/request.ts:60` | ✅ | ❌ | ✅（config） |
| ① | `<env>`：工作目录、worktree、是否 git、平台、日期；`<available_references>` | `session/system.ts:69-105` | ❌ | 日期变 | ✅ |
| ② | `Instructions from: <path>` + 文件内容（远程 URL 同） | `session/instruction.ts:110-169` | ❌ | ❌ | ✅ |
| ③ | `<mcp_instructions>` 每个 server 一段 | `session/system.ts:121-137` | ✅（按 permission 过滤） | ❌ | ✅ |
| ④ | skills 清单（verbose 渲染） | `session/system.ts:107-119` | ✅（skill 被 deny 则整段消失） | ❌ | ✅ |
| ⑥ | `user.system`（客户端传入） | `llm/request.ts:62` | ❌ | 可每轮变 | ❌ |

拼接后的两次加工与落位：

```ts
// llm/request.ts:69-78   插件可改 system 数组（并保持 header 在前）
yield* input.plugin.trigger("experimental.chat.system.transform", { sessionID, model }, { system })

// llm/request.ts:101-112
const messages = isOpenaiOauth || input.isWorkflow
  ? input.messages                                        // OpenAI OAuth / GitLab workflow：走 options.instructions / 专用字段
  : [...system.map(x => ({ role: "system", content: x })), ...input.messages]
```

### ② 段的发现规则（`session/instruction.ts:60-169`）

- 全局：`$XDG_CONFIG_HOME/opencode/AGENTS.md`，不存在则 fallback `~/.claude/CLAUDE.md`（`disableClaudeCodePrompt` 可关）；只取第一个存在的（`:115-120`）。
- 项目级：`AGENTS.md` → `CLAUDE.md` → `CONTEXT.md`（deprecated），每轮从 `ctx.directory` 向上 `findUp` 到 `ctx.worktree`，**第一个命中的文件名在整个祖先链上胜出**（`:122-133`），避免每级目录的 AGENTS.md 都堆进上下文。
- `config.instructions` 支持文件/glob（`:135-150`）与 http(s) URL（`:158-163`，5s 超时，并发 4）。
- `OPENCODE_DISABLE_PROJECT_CONFIG=1` 时只读全局配置目录（`:81-88`）。
- ⚠️ **无缓存**：`system()` 每次调用都重新读文件（并发 8）与抓 URL，而调用点在 `prompt.ts:1260`（while 循环内）⇒ **每个 step 都会读一遍 AGENTS.md**。这是 V1 上下文组装里最稳定的一个额外 IO 点。

---

## 4. Agent 如何决定上下文

`Agent.Info`（`agent/agent.ts:30-55`）逐字段：

| 字段 | 影响上下文的方式 |
|---|---|
| `prompt` | system ⑤ 段；未设则用按模型选的 provider prompt（`llm/request.ts:60`） |
| `model` / `variant` | 模型与变体 → context window、`usable()`、`options` |
| `temperature` / `topP` / `options` | `chat.params` 默认值（`llm/request.ts:124-131`，`mergeDeep` 合并模型与 agent 选项） |
| `permission` | ①工具集（`llm/request.ts:210-216` 的 `Permission.disabled`）②MCP 段是否注入（`system.ts:122-125`）③skills 段是否存在 ④`task` 工具描述里的**可用子 agent 列表**（`tool/registry.ts:265-278` 用 `Permission.evaluate("task", name, agent.permission)` 过滤） |
| `steps` | 单次 prompt 内最大 step；到顶注入 `MAX_STEPS_PROMPT`（`prompt.ts:1179,1281`） |
| `mode` | `primary/subagent/all`：能否当默认 agent（`agent.ts:328-340` 拒绝 subagent/hidden）、能否被 `task` 调用 |
| `hidden` | 不出现在给模型的 agent 列表、不能当默认 agent |
| `description` | 注入主 agent 的工具描述里（"何时该用这个 subagent"，`tool/registry.ts:265-278`） |
| `options` | 透传 provider 的额外参数 |
| `name` | 落库到 `session.agent` / `message.agent` / `message.mode` |

内置 agent（`agent/agent.ts:140-265`）：

| agent | mode | prompt | 权限特征 | 实际用途 |
|---|---|---|---|---|
| `build` | primary | 无（走 provider prompt） | `question: allow`、`plan_enter: allow` | 默认 |
| `plan` | primary | 无 | `edit: { "*": "deny", ".opencode/plans/*.md": allow }` 等 | 计划模式 |
| `general` | subagent | 无 | `todowrite: deny` | 多步/可并行任务 |
| `explore` | subagent | `PROMPT_EXPLORE` | `"*": "deny"` + grep/glob/list/bash/webfetch/websearch/read | 只读探索 |
| `compaction` | primary/hidden | `PROMPT_COMPACTION` | `"*": "deny"` | 压缩摘要（`compaction.ts:358`） |
| `title` | primary/hidden | `PROMPT_TITLE`（temperature 0.5） | `"*": "deny"` | 会话标题：`prompt.ts:193-253`，`system: []`、`tools: {}`、`small: true`，上下文只取到第一条真实 user 消息为止；仅在"无 parentID + 标题仍默认 + 只有一条真实 user 消息"时触发 |
| `summary` | primary/hidden | `PROMPT_SUMMARY` | `"*": "deny"` | 定义了但 V1 未见 `agents.get("summary")` 调用点 |

`permission` 的默认基线（所有内置 agent 共享，`agent.ts:119-138`）：

```ts
Permission.fromConfig({
  "*": "allow",
  doom_loop: "ask",
  external_directory: { "*": "ask", ...白名单: "allow" },   // 白名单：truncation glob / $TMPDIR/opencode / skill dirs / reference dirs
  question: "deny", plan_enter: "deny", plan_exit: "deny",
  read: { "*": "allow", "*.env": "ask", "*.env.*": "ask", "*.env.example": "allow" },
})
```

config 覆盖（`:267-294`）与 `disable: true`（`:268-271`）也在这一步生效；每条规则的合并是 `Permission.merge`（后写覆盖）。

**核心结论：agent 是消息级、不是会话级。**

```ts
// prompt.ts:470-477   写 user 消息时记录 agent
const userMsg = { id, sessionID, role: "user", agent: input.agent, model: {...} }
// prompt.ts:1170     循环里从最新 user 消息取
const agent = yield* agents.get(lastUser.agent)
```

所以同一会话历史里可以混有多个 agent 产出的 assistant 消息（每条都有 `agent` 字段），`session/reminders.ts:37` 就是靠它判断"上一轮是不是 plan"来决定注入 `BUILD_SWITCH`。

---

## 5. 历史 → 上下文的读取与裁剪

### 5.1 `filterCompacted`（`session/message-v2.ts:525-576`）—— 围绕最后一次压缩重排

```
压缩前：[ ...老历史..., compaction-user, summary-assistant, ...tail..., continue-user, ...新对话... ]
压缩后：[ compaction-user, summary-assistant, ...tail..., continue-user, ...新对话... ]
```

- "某次压缩已完成"的判定：该 user message 对应的 assistant 满足 `summary && finish && !error`（`:545-546`）；
- `tail_start_id` 指向要保留的尾部起点（`:538-541`）；
- 末尾按 `[compaction, summary, tail..., 后续]` 重排（`:568-573`）；
- 条件不成立时退化为原顺序返回。

补充两点方向性事实：

- `stream()` 产出的是**最新 → 最旧**的顺序（分页 50 条、页内正序后再从页尾往前 push，`message-v2.ts:463-464, 485-488`），所以上面的扫描是"从最新往前扫"；
- **V1 没有任何"保留最近 N 条"的硬编码**，保留边界完全由 compaction 写入的 `tail_start_id` 决定。

### 5.2 `latest`（`:586-608`）—— 用时间戳而非数组下标

返回 `{ user, assistant, finished, tasks }`；`tasks` = 尚未被 `finished` 覆盖的 `compaction`/`subtask` part（`:596-600`），这是命令驱动 subagent 的入口。

### 5.3 `toModelMessagesEffect`（`:131-419`）—— 转换规则全表

| 行为 | 位置 | 说明 |
|---|---|---|
| 空 parts 消息丢弃 | `:200` | 不产生空消息 |
| 用户 `file` part | `:216-230` | 仅媒体类（非 `text/plain`、非目录）转发为 file part |
| `compaction` → `"What did we do so far?"` | `:232-237` | 压缩请求的可读形式 |
| `subtask` → `"The following tool was executed by the user"` | `:238-243` | 命令驱动 subagent 占位 |
| **跨模型降级** | `:249, 287, 326, 367-380` | 当前模型 ≠ 消息记录模型时：丢 `providerMetadata`/`callProviderMetadata`，**reasoning 降级为普通 text** |
| 已 prune 的工具输出 | `:297-299` | 替换为 `"[Old tool result content cleared]"` |
| 工具输出截断 | `:299` | `truncateToolOutput(output, options?.toolOutputMaxChars)` —— ⚠️ **V1 主链路从不传该 option**（`prompt.ts:1262`、`compaction.ts:219`、`prompt.ts:224` 都只传 `(msgs, model)`），所以**转换层实际不截断**；真正的截断发生在**工具执行时**（`tool/truncate.ts:14-17,77-81`：默认 2000 行 / 50KB，可用 `tool_output` 配置） |
| 中断/挂起的工具调用 | `:353-364` | 补成 `output-error` + `"[Tool execution was interrupted]"`，防止 Anthropic 悬空 `tool_use` |
| 出错消息 | `:252-260` | 除"被中断但有内容"外整条跳过 |
| 媒体出 tool result | `:302-309, 386-403` | provider 不支持时抽成**一条额外 user 消息**（`SYNTHETIC_ATTACHMENT_PROMPT`） |
| 全是 `step-start` 的消息 | `:412` | 丢弃 |
| Anthropic 空 text 分隔符 | `:266-288` | 有签名 reasoning 时空 text 写成 `" "` 以保证回放签名位置正确 |

两个容易踩的点：

- **`synthetic` 标记不参与过滤**：只有 `part.ignored` 与空字符串会被跳过（`:210`）。也就是说 read 工具回显的 `"Called the Read tool with the following input: …"`、plan 模式提醒、`<system-reminder>` 里的邻近 AGENTS.md、后台子 agent 结果、命令触发语等 synthetic 文本，**全都真实占用上下文窗口**。
- `stripMedia` 参数同样在 V1 主链路未被使用（只有 `message-v2.ts:421-426` 的 promise 包装暴露它）。

### 5.4 动态指令注入：`Instruction.resolve`（`:179-221`）

除了 §3 的静态 AGENTS.md 段，V1 还有一条**按需注入**路径：read 工具读取某个文件时，会把该文件向上途经的 `AGENTS.md`/`CLAUDE.md`/`CONTEXT.md` 内容以 `<system-reminder>` 形式追加到**本次 tool output** 里（调用点 `tool/read.ts:300, 355-357`）。

- 去重粒度是"每条 assistant message 一次"：`state.claims: Map<MessageID, Set<string>>`（`instruction.ts:70-77`）；
- 由 `Instruction.clear(messageID)` 在每轮结束时清空（`prompt.ts:1331`，另有 `:691` 的 finalizer）。

这意味着**同一份 AGENTS.md 可能同时以"system 段"和"tool result 里的 system-reminder"两种形态进入上下文**，做上下文预算估算时要把这部分算进去。

---

## 6. 上下文膨胀治理（三道闸）

### 闸 1：`prune` —— 擦除旧工具输出（`session/compaction.ts:273-317`）

```ts
export const PRUNE_MINIMUM = 20_000   // :28  可擦总量超过才动手
export const PRUNE_PROTECT = 40_000   // :29  最近 40k token 的工具输出不动
```

- 从最后往前扫；`turns < 2` 跳过（最近两轮不动，`:291`）；遇到 `summary` 停止（`:292`）；跳过 `PRUNE_PROTECTED_TOOLS`（实际值 `["skill"]`，`:28-31,297`）与已 compacted 的（`:298`）；
- 累计超过 `PRUNE_PROTECT` 的部分打 `part.state.time.compacted = Date.now()`（`:311`）；
- 下次转模型时被替换成 `[Old tool result content cleared]`（`message-v2.ts:297-299`）；
- 默认关闭（`cfg.compaction.prune`，`:275`）；循环结束后异步触发（`prompt.ts:1338`）。

### 闸 2：溢出判定（`session/overflow.ts:8-33`）

```ts
const COMPACTION_BUFFER = 20_000
usable = model.limit.input ? max(0, limit.input - reserved)
                          : max(0, context - maxOutputTokens(model))
reserved = cfg.compaction.reserved ?? min(COMPACTION_BUFFER, maxOutputTokens(model))
isOverflow = (tokens.total || input+output+cache.read+cache.write) >= usable
```

两处触发点：

1. **循环入口**（`prompt.ts:1160-1168`）→ `compaction.create({ auto: true })` 并 `continue`；
2. **流式 `step-finish`**（`session/processor.ts:491-496`）→ 置 `ctx.needsCompaction = true` → `Stream.takeUntil(() => ctx.needsCompaction)`（`:658`）提前结束本轮流 → `process` 返回 `"compact"`（`:693`）→ 循环调用 `compaction.create({ auto: true, overflow: !handle.message.finish })`（`prompt.ts:1320-1328`）。
3. **provider 报错兜底**（`processor.ts:621-632`）：收到 `ContextOverflowError` 时同样置 `needsCompaction`；若 `auto === false` 且当前不是 summary 轮，则直接置错并转 idle。

`cfg.compaction.auto === false` 或模型 `limit.context === 0` 时永不触发（`overflow.ts:28-29`）。

### 闸 3：尾部保留 + 摘要（`session/compaction.ts:223-269, 319-468`）

`select()` 决定"哪些进摘要、哪些原样保留"：

- `cfg.compaction.tail_turns` 限制扫描的轮数（`:228-233`）；
- `preserveRecentBudget` 给出保留预算（`:115-120`：`cfg.compaction.preserve_recent_tokens ?? clamp(usable × 0.25, 2_000, 15_000)`），从最近一轮往前累加（`:235-248`）；
- 预算不够时 `splitTurn` 尝试拆分单轮（`:250-257`）；
- 返回 `{ head, tail_start_id }`：**head 送摘要，tail 原样保留**（`:264-268`）。

`processCompaction()` 摘要流程：

1. 用隐藏 agent `compaction`（`:358`）及其 model，未设则沿用触发消息的模型；
2. 跳过历史中已压缩部分（`completedCompactions` / `hidden`，`:364-368`）；
3. 带上 `previousSummary` 形成滚动摘要（`:366, 384`）；
4. 插件钩子 `experimental.session.compacting`（注入 context / 替换 prompt）、`experimental.chat.messages.transform`（`:373-379`）；
5. 生成 **`user` 消息（带 `compaction` part + `tail_start_id`）+ `assistant` summary 消息**（`:394` 附近，`summary: true`）；
6. `experimental.compaction.autocontinue`（`:501`）决定是否自动继续（后续以 synthetic user 消息驱动，`:537`）。

摘要请求本身的形态值得注意：

- `system: []` + `tools: {}` + `agent = "compaction"` ⇒ 最终 system 就是 `agent/prompt/compaction.txt`（模板正文与 `buildPrompt` 在 `packages/core/src/session/compaction.ts:16-55, 160-174`）；
- 只发**一条 user 消息**，内容 = 摘要模板 + 序列化后的 head 会话；
- `serialize()`（`compaction.ts:54-85`）把历史压成纯文本：`[User]` / `[Assistant]` / `[Assistant reasoning]` / `[Assistant tool call]: name(input)` / `[Tool result]`，单段超 2000 字符截断（`:30, 51-52`），已 prune 的 tool result 显示 `[Old tool result content cleared]`。

### 相关配置项汇总

| 配置 | 默认 | 作用 |
|---|---|---|
| `compaction.auto` | 开 | 关掉则永不自动压缩 |
| `compaction.prune` | **关** | 开启旧工具输出擦除 |
| `compaction.reserved` | `min(20000, maxOutput)` | 为输出预留的 token |
| `compaction.tail_turns` | 未设（按预算自动） | 至少保留最近 N 轮 |
| `compaction.preserve_recent_tokens` | `clamp(usable×0.25, 2k, 15k)` | 摘要时保留的近期历史预算 |
| `snapshot` | 开 | 关掉则不产 patch/snapshot |
| `subagent_depth` | 1 | 子 agent 嵌套层数 |
| `experimental.continue_loop_on_deny` | 未设（=拒绝即停） | 工具被拒后是否继续循环 |
| `agent.<name>.steps` | 未设 | 单轮最大 step |
| `OPENCODE_DISABLE_AUTOCOMPACT` | — | 全局关自动压缩（实现方式是强制 `compaction.auto = false`，`config/config.ts:593-596`） |
| `OPENCODE_DISABLE_PRUNE` | — | 全局关 prune（强制 `compaction.prune = false`，`config/config.ts:597-599`） |

> 顺带一条"看起来有其实没有"的字段：`session.time_compacting`（`packages/core/src/session/sql.ts:58`）在 schema/DB/TUI 都存在，但 **V1 代码里没有任何写入点**，"压缩中"状态不会持久化。

---

## 7. Subagent 的上下文

V1 有**两条**触发路径。

### 路径 A：模型调用 `task` 工具（`tool/task.ts`）

```ts
// :104-117  深度检查（沿 parentID 向上数）
const parent = yield* sessions.get(ctx.sessionID)
let current = parent, depth = 0
while (current.parentID) { depth++; current = yield* sessions.get(current.parentID) }
if (depth >= (cfg.subagent_depth ?? 1)) return yield* Effect.fail(...)

// :119-129  权限询问（bypassAgentCheck 可跳过）
yield* ctx.ask({ permission: id, patterns: [params.subagent_type], always: ["*"], ... })

// :131-134  取子 agent 定义
const next = yield* agent.get(params.subagent_type)

// :136-138  task_id 续跑 → 复用旧子会话
const session = params.task_id ? yield* sessions.get(SessionID.make(params.task_id))... : undefined

// :139-172  权限派生 + 建子会话
const childPermission = deriveSubagentSessionPermission({ parentSessionPermission: parent.permission ?? [], subagent: next })
const childToolDenies = [ todowrite deny?, task deny?, experimental.primary_tools deny? ]
const nextSession = session ?? (yield* sessions.create({
  parentID: ctx.sessionID, title: `${description} (@${next.name} subagent)`,
  agent: next.name, permission: [...childPermission, ...childToolDenies...],
}))

// :181-184  模型：子 agent 自己的，否则继承父消息
const model = next.model ?? { modelID: msg.info.modelID, providerID: msg.info.providerID }

// :201-212  只传 prompt 文本
const result = yield* ops.prompt({
  messageID: MessageID.ascending(), sessionID: nextSession.id,
  agent: next.name, model, variant: next.model ? undefined : variant,
  parts: yield* ops.resolvePromptParts(params.prompt),
})

// :224  只回最后一条 text
return result.parts.findLast((item) => item.type === "text")?.text ?? ""
```

### 路径 B：命令驱动（`session/prompt.ts:1439-1451` + `:256-430`）

斜杠命令若映射到 subagent，产生的不是文本而是 `subtask` part：

```ts
// prompt.ts:1439-1451
const isSubtask = (agent.mode === "subagent" && cmd.subtask !== false) || cmd.subtask === true
const parts = isSubtask
  ? [{ type: "subtask", agent: agent.name, description: cmd.description ?? "",
       command: input.command, model: {...}, prompt: templateParts.find(p => p.type === "text")?.text ?? "" }]
  : [...uniqueTemplateParts, ...(input.parts ?? [])]
```

主循环通过 `latest().tasks` 发现它并**直接调用同一 task 工具**（`prompt.ts:324-349`），带 `extra.bypassAgentCheck: true`（`:331`）跳过询问。区别：**不经过模型决策**，少一次 LLM 往返。

### 子会话的上下文边界

| 维度 | 子会话 | 证据 |
|---|---|---|
| system ⑤ 段 | **换成子 agent 自己的 prompt** | `task.ts:210` → `llm/request.ts:60` |
| system ①-④ 段 | 由子会话自身的 config/agent/permission 决定 | `session/system.ts:107-137` |
| 对话历史 | **空**（新 session），只有 task 的 prompt 文本 | `task.ts:201-212`；历史由 `filterCompactedEffect(nextSession.id)` 取 |
| 父会话消息 | **完全不可见**（零共享） | 同上 |
| 工具集 | 子 agent 权限 + 强制 deny：`todowrite`、`task`、`experimental.primary_tools` | `task.ts:143-155` |
| 权限继承 | 只透传父的 `external_directory` 与所有 `deny`，其余重新按子 agent 算 | `agent/subagent-permissions.ts:14-27` |
| 模型 | 子 agent `model`，否则继承父消息的 providerID/modelID | `task.ts:181-184` |
| 目录/实例 | 与父**同一实例**（`ctx.directory`） | `session.ts:676-681` |
| 嵌套深度 | `cfg.subagent_depth ?? 1` | `task.ts:104-117` |
| 结果回传（前台） | task 工具返回值（`<task id state="completed"><task_result>…`） | `task.ts:64-79, 224, 341-345` |
| 结果回传（后台） | `synthetic: true` 的 user 消息注入父会话 | `task.ts:227-253` |
| 取消 | 父 abort → 级联取消子会话；后台 `cancelled` **不注入通知** | `task.ts:321-331, 347-357, 356-362` |
| 续跑 | `task_id` 复用同一子会话（历史累积），经 `background.extend` 串行排队 | `task.ts:136-138, 157, 267`；`core/src/background-job.ts:256-290` |
| 执行态生命周期 | `BackgroundJob`（内存态、不持久、随实例目录隔离）；进程重启丢状态但会话记录仍在 | `background/job.ts:21`、`core/src/background-job.ts:113-119` |

---

## 8. 流式事件如何落成 part（决定下一轮上下文）

`session/processor.ts` 的 `handleEvent`（`:278-551`）按 LLM 事件写库：

| 事件 | 落库动作 | 位置 |
|---|---|---|
| `reasoning-start/delta/end` | `reasoningMap[id]` 累积，结束时 `finishReasoning` 写回 | `:280-314`, `:207-214` |
| `tool-input-*` / `tool-call` | 建/更新 `tool` part（`pending`→`running`）；同时做 **doom loop 检测**（见下） | `:315-382` |
| `tool-result` | `completeToolCall` → `state.status = "completed"`（含 `attachments`） | `:383-415`, `:160-185` |
| `tool-error` | `failToolCall` | `:416-420`, `:186-206` |
| `provider-error` | `halt(...)` | `:421-423` |
| `step-start` | `step-start` part（**带 snapshot**） | `:424-433` |
| `step-finish` | ① `snapshot.track()`；② 结束所有 reasoning；③ `usage` → `tokens`/`cost` 写回 assistant 消息；④ 写 `step-finish` part；⑤ 有文件变更则写 `patch` part；⑥ `summary.summarize(...)`（**文件 diff 统计**，异步）；⑦ `isOverflow` → `needsCompaction = true` | `:435-497` |
| `text-start/delta/end` | `currentText` 累积；`text-end` 时经 `experimental.text.complete` 插件加工后写回 | `:500-546` |
| `finish` | 忽略（已由 step-finish 处理） | `:548-549` |

**doom loop 检测**（`processor.ts:29, 331-380`）：`DOOM_LOOP_THRESHOLD = 3`，只统计**当前 assistant 消息内**最后 3 个 tool part，若"同工具 + 同 input"连续三次则发起一次 `doom_loop` 权限询问（默认 `"ask"`）。被拒绝时 `ctx.blocked` 置位（`:200-202`），`process()` 返回 `"stop"`，循环结束（`shouldBreak` 由 `experimental.continue_loop_on_deny` 控制，`:647`）。

**`process()` 的三种返回值**（`:641-697`）决定下一轮动作：`"compact"`（`ctx.needsCompaction`）→ 主循环插入压缩任务；`"stop"`（`ctx.blocked || assistantMessage.error`）→ `break`；其余 `"continue"` → 再走一轮。

注意：`session/summary.ts` 的 `summarize` 是**文件变更统计**（additions/deletions/files，基于 snapshot diff，`summary.ts:82-119`），与"上下文压缩摘要"是两件事。

---

## 9. 插件钩子在上下文管线上的介入点

| 钩子 | 介入时机 | 可改内容 |
|---|---|---|
| `chat.message` | 用户消息落库后 | user message 的 parts |
| `experimental.chat.messages.transform` | 转模型格式前后（`prompt.ts:1255`、`compaction.ts:379`） | `msgs` 数组 |
| `experimental.chat.system.transform` | system 串拼好后（`llm/request.ts:69-78`） | system 段 |
| `chat.params` / `chat.headers` | 参数/请求头定稿（`llm/request.ts:114-146`） | 采样参数、请求头 |
| `tool.definition` | 工具描述发给模型前（`tool/registry.ts:318`） | 工具描述与参数 schema |
| `tool.execute.before` / `after` | 每次工具执行（`session/tools.ts:106-125`） | args / 输出；抛错即中断 |
| `command.execute.before` | 命令展开为 parts 后（`prompt.ts:1460`） | 命令产生的 parts（含 subtask） |
| `experimental.session.compacting` | 摘要生成前（`compaction.ts:373-377`） | 额外 context、替换压缩 prompt |
| `experimental.compaction.autocontinue` | 摘要完成后（`compaction.ts:501`） | 是否自动继续 |
| `experimental.text.complete` | 文本流结束（`processor.ts:530-538`） | 最终文本 |

---

## 10. 上下文预算构成清单

| 内容 | 注入时机 | 随轮次增长 | 控制手段 |
|---|---|---|---|
| agent 人格（⑤） | 每轮 | ❌ | `agent.prompt` |
| 环境信息（①） | 每轮 | ❌ | references 数量 |
| AGENTS.md / instructions（②） | **每 step 重读** | ❌ | 文件大小、`OPENCODE_DISABLE_PROJECT_CONFIG` |
| MCP instructions（③） | 每轮 | ❌ | MCP server 数、permission |
| skills 清单（④） | 每轮 | ❌ | skill 数量（verbose） |
| 工具描述 + schema | 每次请求 | ❌（但总量大） | agent permission、`tool.definition` |
| 用户消息 + 附件 | 累积 | ✅ | 附件媒体是大头（`stripMedia` 可降级） |
| assistant 文本 | 累积 | ✅ | 压缩 |
| reasoning | 累积 | ✅ | 压缩；跨模型时降级为 text |
| 工具入参 + 结果 | 累积 | ✅ **最大头** | prune（40k/20k）、输出截断 |
| 摘要（summary 消息） | 每次压缩替换一段历史 | 阶梯式 | compaction 预算 |
| `MAX_STEPS_PROMPT` | `step >= agent.steps` | 每轮至多一次 | `agent.steps` |
| plan/build 提醒（synthetic） | plan 相关轮 | 每轮 | `reminders.ts` |

---

## 11. 对多智能体平台的实操建议

1. **每 step 全量重读 DB + 重算 system + 重读 AGENTS.md**：会话越长、step 越多，固定开销越大。压测时要把这部分计入；instructions 文件不要放超大文件（**V1 对它没有任何大小/token 上限**，`instruction.ts:91-93` 只是 `readFileString`）。
2. **agent 是消息级属性**：平台若要求"一个会话绑定一个智能体"，需在控制面固定 `agent`；若要多智能体协作，V1 原生支持（同会话换 agent + `task` 派生 subagent）。
3. **subagent 计费/统计必须区分**：子会话是独立 `session` 行（`parent_id` 非空）、独立 token 统计，但存在同一个库里；列表用 `GET /session?roots=true`（`session.ts:985-987` 的 `isNull(parent_id)`）过滤。`task_id` 续跑复用同一子会话，不能按"一次 task 调用=一个会话"计数。
4. **上下文治理只有 prune 与 compaction 两个旋钮**（外加 `agent.steps` 的重置点）：`compaction.prune` 默认关闭，`tail_turns` 未设时按预算自动算；要兜底用户预算就显式配置 `compaction.reserved` / `tail_turns` / `preserve_recent_tokens`。
5. **synthetic 文本会真实占用上下文**（§5.3 的两个"容易踩的点"）：read 工具回显、plan 提醒、`<system-reminder>`、后台子 agent 结果等都不会被过滤。做上下文预算或内容审计时，**不要假设"只有用户输入和模型输出进上下文"**。
6. **删会话是递归且静默的**：`Session.remove` 先 `cancelBackgroundJobs` 再递归删子会话，整段包在 try/catch 里只记日志（`session.ts:606-627`）；清理租户数据时不要只信接口返回。

---

## 12. 附录：源码索引

| 主题 | 位置 |
|---|---|
| 主循环（时序、agent 解析、max steps） | `packages/opencode/src/session/prompt.ts:1085-1339` |
| 写 user 消息（记录 agent/model） | `packages/opencode/src/session/prompt.ts:455-490` |
| system ①-④ 段组装 | `packages/opencode/src/session/prompt.ts:1257-1271` |
| system ⑤⑥ 段与最终顺序、system.transform、messages 拼装 | `packages/opencode/src/session/llm/request.ts:56-112` |
| 环境 / skills / MCP 段 | `packages/opencode/src/session/system.ts:69-137` |
| AGENTS.md 等指令发现 | `packages/opencode/src/session/instruction.ts:60-169` |
| 历史重排 | `packages/opencode/src/session/message-v2.ts:525-576` |
| 取最新 | `packages/opencode/src/session/message-v2.ts:586-608` |
| 历史 → 模型消息 | `packages/opencode/src/session/message-v2.ts:131-419` |
| prune 阈值与实现 | `packages/opencode/src/session/compaction.ts:28-29, 273-317` |
| 溢出判定 | `packages/opencode/src/session/overflow.ts:8-33` |
| 尾部选择 | `packages/opencode/src/session/compaction.ts:223-269` |
| 摘要生成 | `packages/opencode/src/session/compaction.ts:319-468` |
| 流事件落库 | `packages/opencode/src/session/processor.ts:278-551` |
| 工具集过滤 | `packages/opencode/src/session/tools.ts:92-134`、`session/llm/request.ts:210-216`、`tool/registry.ts` |
| agent 定义与内置 agent | `packages/opencode/src/agent/agent.ts:30-55, 119-265, 267-344` |
| task 工具（subagent） | `packages/opencode/src/tool/task.ts`（全文 371 行） |
| subagent 权限派生 | `packages/opencode/src/agent/subagent-permissions.ts:14-27` |
| 命令驱动 subtask | `packages/opencode/src/session/prompt.ts:256-430, 1439-1451` |
| 后台任务注册表 | `packages/core/src/background-job.ts`、`packages/opencode/src/background/job.ts:21` |
| plan/build 提醒注入 | `packages/opencode/src/session/reminders.ts:15-90` |
| 最大 step 提示词 | `packages/core/src/session/runner/max-steps.ts` |
| 表结构 | `packages/core/src/session/sql.ts:22-98` |
| 会话删除（递归 + 静默失败） | `packages/opencode/src/session/session.ts:606-627` |
| 列表过滤 roots/project/directory | `packages/opencode/src/session/session.ts:546-553, 960-1008` |
| 会话文件 diff 统计（非上下文压缩） | `packages/opencode/src/session/summary.ts:82-119` |
