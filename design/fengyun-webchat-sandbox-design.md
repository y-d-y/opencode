# 风云网 WebChat 多智能体平台：多用户沙箱隔离技术方案

> 依据源码：`E:\0_GITHUB\opencode`（opencode `1.18.34`，Bun + TypeScript + Effect）。
> 本文所有结论都给出源码位置（`文件:行号`），便于评审时逐条核对。

---

## 0. 结论摘要（TL;DR）

| 问题 | 现状根因 | 结论 |
|---|---|---|
| 多用户文件系统互相可见 | 一个 agent 一个 `opencode serve` 进程，`cwd` 固定为智能体 home，所有用户共用同一个目录 | 隔离边界必须是 **OS 级**：每用户一个进程 + mount/pid/user namespace，用户目录互相不可见 |
| 会话记录共享 | 会话存在 **每个 app 目录自己的** SQLite 里，与用户无关 | 按用户切 `XDG_DATA_HOME`（或显式 `OPENCODE_DB`），一用户一库 |
| 工具执行无隔离 | opencode 明确声明 bash **不是沙箱**（`specs/v2/session.md:204`），`external_directory` 只是"越界即询问" | 沙箱化**整个 opencode 进程**，而不是只包 bash 工具；permission 仅作纵深防御 |

**推荐架构（一句话）**：把 `apps/<agent>` 的"一个智能体一个进程"升级为 **`(tenant_id, agent_name)` 一个沙箱化 opencode 进程**；沙箱用 `bwrap`（bubblewrap，本机、非特权）或 `runc/nsjail` 落地，进程内只 bind 该用户自己的目录 + 只读的智能体模板；控制面（`api/chat_server.py`、`mgr/mgr_server.py`）升级为"沙箱管理器 + 路由 + 生命周期"，负责 `(user, agent) → endpoint` 的映射、池化、空闲回收、权限代答与越权校验。

三条可选路线（后文 3 节详述）：

- **A. 单进程多目录（`x-opencode-directory`）**：零进程开销，但**只能解决"会话/目录串味"，不是安全边界**，必须修正 project id 碰撞问题。适合做 Phase 0 救火。
- **B. 每用户沙箱进程（推荐起步）**：`bwrap` + 每用户 XDG/DB + 每沙箱口令；单 Pod 内即可落地，隔离强度"进程 + 文件系统可见性 + UID/namespace"。
- **C. 每用户 Sandbox Pod（推荐终态）**：K8s `agent-sandbox`（`Sandbox`/`SandboxWarmPool` + `RuntimeClass`: gVisor/Kata）+ `NetworkPolicy` + 资源配额；天然解决 CPU/内存/网络三类硬隔离。

---

## 1. 现状与问题定位（源码级）

### 1.1 今天到底共享了什么

你的 `start_agent_process()` 等价于：

```
cwd            = apps/<agent>            # 每个用户都是这一个目录
XDG_CACHE_HOME = apps/<agent>/local/cache
XDG_STATE_HOME = apps/<agent>/local/state
XDG_DATA_HOME  = apps/<agent>/local/share   # ← 会话 SQLite 在这里
OPENCODE_CONFIG= apps/<agent>/opencode.json # ← 所有用户同一份配置
```

因此四类共享同时存在：

1. **文件系统上下文**：`cwd` 唯一，工具的读写根、`git`、LSP 索引、`.env`、生成物全部串在一起。
2. **会话记录**：同一个 SQLite 文件、同一张 `session` 表。
3. **进程内状态**：权限审批（`approved` 列表）、LSP/watcher、插件实例、MCP 连接、模型会话池全是进程级共享。
4. **凭据与出网身份**：同一个 `OPENCODE_SERVER_PASSWORD`（你们甚至可能没设）、同一份 `auth.json`、同一个出口 IP 与 TLS 配置（`NODE_TLS_REJECT_UNAUTHORIZED=0`）。

### 1.2 关键源码证据

**（1）`serve` 的目录是"每请求"决定的，不是启动时固定的** —— 这是双刃剑：

```ts
// packages/opencode/src/cli/cmd/serve.ts:10-12
// Server loads instances per-request via x-opencode-directory header — no
// need for an ambient project InstanceContext at startup.
instance: false,
```

```ts
// packages/opencode/src/server/routes/instance/httpapi/middleware/workspace-routing.ts:86-88
function defaultDirectory(request, url) {
  return url.searchParams.get("directory") || request.headers["x-opencode-directory"] || process.cwd()
}
```

```ts
// .../middleware/instance-context.ts:27-34
const route = yield* WorkspaceRouteContext
const ctx = yield* store.load({ directory: decode(route.directory) })   // 每请求装载实例
return yield* effect.pipe(Effect.provideService(InstanceRef, ctx), ...)
```

**（2）进程内"多租户"只有 `directory` 这一层键，且**没有归属校验**：

```ts
// packages/opencode/src/project/instance-store.ts:43,108-124
const cache = new Map<string, Entry>()                 // 全局单例，无容量/TTL
const directory = FSUtil.resolve(input.directory)      // realpath 归一化后作为唯一键
```

```ts
// packages/opencode/src/project/instance-context.ts:18-24
export function containsPath(filepath, ctx) {
  if (FSUtil.contains(ctx.directory, filepath)) return true
  if (ctx.worktree === "/") return false      // 非 git 项目：边界只有 directory
  return FSUtil.contains(ctx.worktree, filepath)
}
```

**（3）会话记录的隔离键是 `(project_id, directory)`，而 `project_id` 来自 git 身份 —— 跨用户会碰撞**：

```ts
// packages/core/src/project.ts:110-122
const resolve = Effect.fn("Project.resolve")(function* (input) {
  const repo = yield* git.repo.discover(input)
  if (!repo) return { id: ID.global, directory: path.parse(input).root, vcs: undefined }  // ← 非 git 全部塌缩成 "global"
  const previous = yield* cached(repo.commonDirectory)
  const id = (yield* remote(repo)) ?? previous ?? (yield* root(repo))                     // ← hash(git remote) 或 root commit
  ...
})
```

```ts
// packages/core/src/session/sql.ts:26-33
project_id: text().$type<ProjectV2.ID>().notNull().references(() => ProjectTable.id, ...),
workspace_id: text().$type<WorkspaceV2.ID>(),
directory: DatabasePath.directoryColumn().notNull(),
```

含义：
- 两个用户各自 `git clone` 同一个仓库（且带 origin）→ **project id 完全相同**；
- 非 git 目录 → 全部落到 `"global"`，靠 `directory` 列区分；
- 所以"按目录隔离会话"只在单进程内、且目录唯一时才成立。**真正的会话隔离必须靠独立的 DB 文件**。

**（4）`/event` SSE 是按 directory 过滤的（好消息）**：

```ts
// packages/opencode/src/server/routes/instance/httpapi/handlers/event.ts:35-39
Stream.filter((event) =>
  event.location?.directory === instance.directory &&
  (event.location.workspaceID === undefined || event.location.workspaceID === workspaceID))
```

**（5）但是"用 sessionID 直接访问"会以 session 记录的目录为准，绕过请求头**：

```ts
// .../middleware/workspace-routing.ts:181-184
return RequestPlan.Local({
  directory: session?.directory || defaultDirectory(request, url),   // ← session 目录优先
  workspaceID: envWorkspaceID ?? workspaceID,
})
```

只要拿到别人的 `sessionID`，`GET /session/:id/message`、`POST /session/:id/message` 就会在**别人的目录**里执行。sessionID 本身是 `ses_` + 12 位时间 + 14 位随机（`packages/schema/src/identifier.ts:14-29`，约 83 bit 随机），不可猜，但**平台层绝不能把 sessionID 当成"用户提交的普通参数"直接转发**。

补充两条精确边界（已核对 `routeHttpApiWorkspace` 的完整链路）：

- **`x-opencode-directory` 头无法把已存在的会话重定向到别的目录**：`routeHttpApiWorkspace`（`workspace-routing.ts:212-235`）先从 URL 里提取 sessionID（`server/shared/workspace-routing.ts:20-29`，匹配 `^/session/<id>` 的所有子路径），再查库拿 `session.directory`，最后 `directory: session?.directory || defaultDirectory(...)`（`:181-184`）。所以"越权"的前提始终是**攻击者能对别人的 sessionID 发出请求**（即控制面路由失误），而不是外部能靠改 header 指定目录。
- **例外是列表类接口**：`GET /session` 不匹配 sessionID 规则（`isLocalWorkspaceRoute` 里 `GET /session` 单独标记，`:5-9`），它的目录完全由请求头/查询参数决定 —— 这正是 §4.8.2 里"必须显式传 `?directory=`"的原因。

**（6）permission 是"询问式"，不是围栏**：

```ts
// packages/opencode/src/permission/index.ts:67-107（InstanceState 作用域，per-directory 的 pending/approved）
for (const pattern of request.patterns) {
  const rule = evaluate(request.permission, pattern, ruleset, approved)
  if (rule.action === "deny") ...
  if (rule.action === "allow") continue
  needsAsk = true
}
```

```ts
// packages/opencode/src/agent/agent.ts:119-136（默认规则）
const defaults = Permission.fromConfig({
  "*": "allow",
  doom_loop: "ask",
  external_directory: { "*": "ask", ...白名单: "allow" },
  question: "deny",
  read: { "*": "allow", "*.env": "ask", "*.env.*": "ask", "*.env.example": "allow" },
})
```

**（7）bash 工具直接 spawn 宿主 shell，没有沙箱**：

```ts
// packages/opencode/src/tool/shell.ts:293-310
return ChildProcess.make(command, [], { shell, cwd, env, stdin: "ignore", detached: ... })
```

```ts
// packages/opencode/src/tool/shell.ts:416-426（唯一可注入的钩子：环境变量）
const extra = yield* plugin.trigger("shell.env", { cwd, sessionID, callID }, { env: {} })
return { ...process.env, ...extra.env }
```

上游自己也写明了这一点：

```md
<!-- specs/v2/session.md:204 -->
Bash is not sandboxed: the spawned shell runs with the host user's filesystem,
process, and network authority. ... Best-effort scans of absolute command
arguments produce advisory warnings only; they are not sandbox boundaries.
```

**（8）全仓库没有任何 OS 级沙箱实现**：搜 `bwrap|bubblewrap|nsjail|landlock|seccomp|gVisor|firejail` 零命中；`Project.sandboxes` 指的是 git worktree 目录列表（`packages/opencode/src/project/project.ts:90-99`），不是安全沙箱。

**（9）用户可写目录会参与配置/插件发现 —— 等于在进程内执行任意代码**：

```ts
// packages/opencode/src/config/config.ts:420-424
if (!Flag.OPENCODE_DISABLE_PROJECT_CONFIG) {
  for (const file of yield* ConfigPaths.files("opencode", ctx.directory, ctx.worktree)) {
    yield* merge(file, yield* loadFile(file, authEnv), "local")
  }
}
// :438-479 还会扫描沿路 .opencode/ 目录：agent/command/plugin（ConfigPlugin.load(dir) 直接 import 本地 ts）
```

只要用户能在 `cwd` 或其祖先写一个 `opencode.json` / `.opencode/plugin/x.ts`，就能在 opencode 进程内执行代码。多租户下这必须是**硬禁用**（见 4.6）。

**（10）除了 bash 工具，至少还有 4 条"不过权限系统"的执行路径**（沙箱必须覆盖整个进程，而不是只包 bash）：

| 路径 | 位置 | 是否过 permission |
|---|---|---|
| PTY 终端 | `packages/core/src/pty.ts:165-183`、路由 `.../httpapi/handlers/pty.ts:60-82` | ❌ |
| `!command` / 命令模板 | `packages/opencode/src/session/prompt.ts:552-578` | ❌ |
| 插件自带的 `Bun.$` | `packages/opencode/src/plugin/index.ts:167` | ❌ |
| MCP server 进程 | `packages/opencode/src/mcp/*`（配置驱动） | ❌ |
| LSP / formatter / rg / git 子进程 | 各处 `ChildProcessSpawner` | ❌ |

换句话说：**只要你的沙箱只包住 `bash` 工具，上面这些入口都能直接跑出沙箱**。

**（11）默认权限是最宽松的**：`"*": "allow"`（`packages/opencode/src/agent/agent.ts:120`），只有 `external_directory`/`doom_loop`/`*.env` 例外。任何"默认会弹窗问"的假设都不成立，必须显式配置。

**（12）`XDG_*` 与 `Global.Path.*` 是模块导入期求值的常量**（`packages/core/src/global.ts:11-15`，仓库自己在 `packages/opencode/test/preload.ts:1-2` 注明"必须在任何 src 导入之前设置"）。因此**同一进程内按请求切换租户目录是不可能的**——这从机制上决定了"每租户一进程"。

### 1.3 顺带指出的现存问题（建议一并修）

| 位置 | 问题 | 影响 |
|---|---|---|
| 你的 `start_agent_process` | `opencode_dir = app_dir / "opencode"` 后 `cwd=str(opencode_dir)`：指向二进制文件本身，Linux 下 `Popen` 会 `NotADirectoryError` | 若线上能跑通，说明实际目录结构与文档不符；沙箱化后 `cwd` 必须精确等于该用户的挂载点 |
| 上述 env | `NODE_TLS_REJECT_UNAUTHORIZED=0` 全局关闭 TLS 校验 | 多租户下等于允许任意 MITM；应改为注入内网 CA |
| 上述 env | `OPENCODE_PURE=0` + 用户可写目录 ⇒ 外部插件代码在进程内执行 | 沙箱后也要保证插件目录只读 |
| `mgr` 的 start 接口 | 单例进程，无用户维度 | 见 5.2 的扩展接口 |

---

## 2. opencode 提供的"可直接用的隔离原语"（别重复造轮子）

| 原语 | 位置 | 用途 |
|---|---|---|
| `OPENCODE_DB`（绝对路径或相对 data 目录） | `packages/core/src/flag/flag.ts:47`、`packages/core/src/database/database.ts:44-54` | 一用户一库，最直接的会话隔离开关 |
| `XDG_DATA_HOME` / `XDG_*` | `packages/core/src/global.ts:3-14` | data/log/state/cache 整体搬家 |
| `OPENCODE_CONFIG` / `OPENCODE_CONFIG_DIR` | `flag.ts:21-22,63-65`、`config.ts:415-418,432-434` | 用只读的智能体配置替换用户可写配置 |
| `OPENCODE_DISABLE_PROJECT_CONFIG` | `config.ts:420`、`config/paths.ts:27` | 关闭项目级 `opencode.json` 与 `.opencode/` 发现（**多租户必开**） |
| `OPENCODE_PERMISSION`（JSON 规则集） | `config.ts:559-563` | 由控制面强制注入权限规则，用户改不掉 |
| `OPENCODE_SERVER_PASSWORD` / `_USERNAME` | `server/auth.ts:18-19`、`cli/cmd/serve.ts:15-17` | 每沙箱独立口令（Basic Auth），未设置仅打印 warning |
| `x-opencode-directory` / `?directory=` | `workspace-routing.ts:86-88` | 单进程多目录路由（**注意：按沙箱内视角解析路径**） |
| `OPENCODE_WORKSPACE_ID` | `flag.ts:49`、`workspace-routing.ts:66` | 固定工作区身份，且会关闭 remote 代理分支 |
| InstanceState（per-directory ScopedCache） | `packages/opencode/src/effect/instance-state.ts:26-50` | 权限、LSP、watcher 等按目录隔离与自动回收 |
| LocationServiceMap（60min idle TTL） | `packages/core/src/location-services.ts:84-112` | 每目录一整套服务 + 按需回收（但只回收服务层，不回收目录内容） |
| Workspace 抽象 + 插件 adapter | `packages/plugin/src/index.ts:47-66`、`control-plane/adapters/index.ts:37-41` | 上游官方的"把会话放到另一个执行目标"扩展点，见 6.1 |
| 远端 workspace 代理 + 会话 warp | `control-plane/workspace.ts:492-557`、`middleware/workspace-routing.ts:113-146` | 把请求代理到"沙箱里的 opencode"，含 SSE 转发与 Fence 同步 |

**明确没有的**：进程/文件系统/网络的强制隔离、cgroup 配额、目录归属校验、跨租户的 DB 分离、沙箱生命周期管理。这些都必须由你们在控制面与 OS 层补。

---

## 3. 目标模型与选型

### 3.1 先定义三件事

1. **隔离边界（blast radius）**：**用户（tenant）**。同用户的多智能体可以共处一个沙箱（代价可接受），跨用户必须互不可见。
2. **实例粒度**：`(tenant, agent)` 一个 `opencode serve` 进程（默认，改动最小、与现有架构同构）；规模化后可选"每用户一个进程 + 目录级多实例"。
3. **强制手段**：OS 级（namespace + 只挂载自己的目录 + 独立 UID/口令），opencode 的 permission 只做纵深防御。

### 3.2 三种方案对比

| 维度 | A. 单进程多目录 | B. 每用户沙箱进程（推荐起步） | C. 每用户 Sandbox Pod（推荐终态） |
|---|---|---|---|
| 落地方式 | 一个 `serve` + `x-opencode-directory` | `bwrap`/`nsjail` 包住每个 `serve` | K8s `Sandbox` CRD + `RuntimeClass`(gVisor/Kata) |
| 文件系统隔离 | ❌ 仅"不同目录"，进程能读全盘 | ✅ mount ns 只挂自己的目录 + 只读模板 | ✅ 独立容器文件系统 + PVC |
| 会话隔离 | ⚠️ 需修 project id 碰撞；同库不同目录 | ✅ 独立 DB 文件 | ✅ 独立 PVC |
| 进程/信号隔离 | ❌ 共享 PID/权限状态 | ✅ `--unshare-pid` | ✅ 独立 Pod |
| 资源配额 | ❌ | ⚠️ 仅 ulimit/prlimit | ✅ requests/limits + QoS |
| 网络隔离 | ❌ | ⚠️ 共享 Pod netns（loopback 互达，靠口令） | ✅ NetworkPolicy / 独立 netns |
| 启动开销 | 0 | ~百毫秒 | 秒级（用 WarmPool 缓解） |
| 改造量 | 小 | 中 | 大（需要集群能力配合） |
| 适用 | 救火/开发环境 | 生产起步 | 生产终态、合规场景 |

> 结论：**先做 B，把 C 作为 6-12 个月目标**。A 只在 Phase 0 作为过渡，且必须明确"它不是安全边界"。

### 3.3 目标架构

```
外部调用方（webchat 前端）
    │  POST /chat  { user_id, agent_name, session_id?, message }
    ▼
┌───────────────────────────────────────────────────────────────┐
│ api/chat_server.py  :18001          调度服务（控制面）          │
│  · 鉴权：user_id ← 可信身份（不信任客户端字段）                 │
│  · 归属校验：session_id 必须属于该 user/agent                   │
│  · 路由：(user_id, agent_name) → SandboxHandle{endpoint, token} │
│  · SSE 透传 + 权限代答（permission.asked → 策略引擎 → reply）    │
└───────────────┬───────────────────────────────────────────────┘
                │ ensure(user,agent) / 事件回写
                ▼
┌───────────────────────────────────────────────────────────────┐
│ mgr/mgr_server.py :18000         智能体配置 + 沙箱管理器         │
│  · agent CRUD（原有）                                           │
│  · sandbox: ensure/start/stop/destroy/list/stats（新增）        │
│  · 池化 + 空闲回收 + 端口分配 + 日志/配额                        │
└───────────────┬───────────────────────────────────────────────┘
                │ fork/exec（bwrap 包裹）
                ▼
┌───────────────────────────────────────────────────────────────┐
│ 沙箱内（mount/pid/uts/ipc ns，独立 HOME/XDG/DB，独立口令）       │
│  /opt/agent   (ro)  ← 智能体模板：opencode 二进制+prompt+skills  │
│  /workspace   (rw)  ← 该用户该智能体的工作区（cwd）              │
│  /home/tenant (rw)  ← HOME/.config/opencode                     │
│  /var/lib/tenant(rw)← XDG_DATA_HOME：opencode.db / log / repos   │
│  opencode serve --hostname 127.0.0.1 --port <per-sandbox>        │
└───────────────────────────────────────────────────────────────┘
      （阶段 C：这一整块换成独立 Sandbox Pod，控制面改用 HTTP 直连/Router）
```

---

## 4. 详细设计（方案 B）

### 4.1 磁盘布局

> **方案二（每 (租户, 智能体) 一进程）的具体示例、三视角对照与运行期落点，见 §4.10。** 本节只给骨架；§4.10 给出可直接照抄的完整目录树。
```
/var/sandbox/tenants/<tenant_id>/            # 0700，属主 = 该租户 UID
├── home/                                    # HOME（沙箱内 /home/tenant）
│   ├── .config/opencode/                    # XDG_CONFIG_HOME/opencode（用户可写，只影响自己）
│   ├── .local/share/opencode/               # XDG_DATA_HOME/opencode
│   │   ├── opencode.db                      # ★ 该租户独有的会话库
│   │   └── log/  repos/  snapshot/  storage/
│   ├── .local/state/opencode/               # XDG_STATE_HOME（plugin-meta.json、Flock 根）
│   └── .cache/opencode/                     # XDG_CACHE_HOME（bin/、skills/、models.json）
├── agents/<agent_name>/
│   ├── ws/                                  # ★ 用户工作区：沙箱内 /workspace，cwd
│   └── run/                                 # pid / port / 状态文件（宿主侧）
└── meta.json                                # 租户元数据：UID、配额、创建时间

/opt/agents/<agent_name>/                     # 平台侧只读模板（从 mgr/agent_template 物化）
├── opencode                                  # 二进制（190MB，硬链接/只读挂载以省空间）
├── opencode.json                             # ★ 由 mgr 下发，只读
├── config/                                   # OPENCODE_CONFIG_DIR：agent/command/plugin/skills
├── prompt/ …, .agents/skills/ …              # 只读
└── skills/ -> /packages/skills（PVC 子路径，只读）
```

> 目录归属说明：`project_id` 相同时（例如多个目录都 clone 了同一个仓库），`snapshot/`、`worktree/` 会落在 `$XDG_DATA_HOME/opencode/<project.id>/` 下。因为 data 目录已按租户（或按智能体）隔离，这里天然不会跨沙箱共享。
> 另外，**非 git 工作区不会启用 snapshot**（`packages/opencode/src/snapshot/index.ts:168-169`），所以"智能体自己写脚本"这类非代码场景根本不会创建影子库。

要点：
- **模板与用户数据彻底分离**：模板只读挂载，用户只能写 `home/` 与 `agents/*/ws/`。
- 一个 agent 模板可被 N 个租户以 ro 方式共享，磁盘上只有一份 190MB 二进制（`--ro-bind` 同一源路径即可）。
- 若需"每用户可安装依赖"，在 `ws/` 内允许；不要给 `/opt` 写权限。

#### 4.1.1 选方案二时，把 `home/` 下沉到"智能体级"（推荐）

上面这棵树是"**租户级 `home/` + 每 agent 一个 `ws/`**"。它适合 §4.8 的合并进程（一个用户一个进程）。**既然选了方案二（每 (租户, 智能体) 一进程），建议改成 `agents/<agent>/home/`**：

| | 租户级 `home/`（你贴的版本） | **智能体级 `home/`（方案二推荐）** |
|---|---|---|
| 会话库 | 该租户所有智能体共用 `opencode.db` | **每个 (租户,智能体) 一个库**，物理隔离 |
| `$XDG_CONFIG_HOME/opencode`（全局配置目录） | 所有智能体共享 ⇒ A 的全局配置/插件会影响 B | 各自独立 |
| `GET /session` 漏传 `?directory=` | 会串到同租户其他智能体 | 该库里只有本智能体 |
| `project_id` 塌缩（非 git ⇒ `"global"`） | 在同一库里混所有智能体 | 只在本库内，无影响 |
| 进程 env（`OPENCODE_CONFIG`/`_CONFIG_DIR`/`_PERMISSION`） | 由各进程自带 ⇒ 已经按 agent 不同 | 同左（这是方案二最大的便利） |
| 磁盘 | 省一份缓存 | 每个 agent 多几 MB 缓存（预装 `rg` + 关闭 models fetch 后可忽略） |
| 销户 | 删租户目录 | 删 `agents/<agent>/` 即可按 agent 下线 |

结论：**方案二 + 智能体级 `home/`**，语义最干净，也和"一进程一配置"完全对齐。§4.10 的所有示例都按这个版本来写。

### 4.2 启动参数（改造后的 `start_agent_process`）

> 下面这段按"租户级 `home/`"写，便于和 §4.1 的骨架对照；若采用 §4.1.1 推荐的**智能体级 `home/`**，只需把 `home` 路径换成 `agents/<agent>/home`、并补上 `OPENCODE_DISABLE_PROJECT_CONFIG=1`，其余不变 —— 完整可照抄版本见 **§4.10.1 视角 C**。

```python
def start_agent_process(tenant_id: str, app_name: str, port: int) -> subprocess.Popen:
    """启动一个租户的一份沙箱化 opencode serve"""
    tenant = TENANTS / tenant_id                 # /var/sandbox/tenants/<id>
    template = AGENT_TEMPLATES / app_name        # /opt/agents/<agent>
    ws = tenant / "agents" / app_name / "ws"     # 用户工作区（宿主视角）
    log_dir = LOGS / tenant_id / app_name
    log_dir.mkdir(parents=True, exist_ok=True)
    log_file = log_dir / f"opencode_{port}.log"

    # ★ 每沙箱独立口令：只发给控制面，禁止复用
    token = secrets.token_urlsafe(32)

    env = {
        "PATH": "/usr/local/bin:/usr/bin:/bin",
        "TZ": "Asia/Shanghai",
        "HOME": "/home/tenant",                  # 沙箱内视角
        "XDG_CACHE_HOME": "/home/tenant/.cache",
        "XDG_STATE_HOME": "/home/tenant/.local/state",
        "XDG_DATA_HOME": "/home/tenant/.local/share",
        "XDG_CONFIG_HOME": "/home/tenant/.config",

        # —— 会话/配置隔离（关键）——
        "OPENCODE_DB": "/home/tenant/.local/share/opencode/opencode.db",
        "OPENCODE_CONFIG": "/opt/agent/opencode.json",       # 只读模板配置
        "OPENCODE_CONFIG_DIR": "/opt/agent/config",          # .opencode 风格目录（agent/command/skill）
        "OPENCODE_DISABLE_PROJECT_CONFIG": "1",              # ★ 禁止用户工作区注入配置/插件
        "OPENCODE_PERMISSION": json.dumps(PERMISSION_RULESET),  # ★ 控制面强制权限
        "OPENCODE_SERVER_PASSWORD": token,
        "OPENCODE_DISABLE_MODELS_FETCH": "1",
        "OPENCODE_DISABLE_AUTOUPDATE": "1",
        "OPENCODE_DISABLE_LSP_DOWNLOAD": "1",
        "OPENCODE_DISABLE_EMBEDDED_WEB_UI": "1",
        "OPENCODE_PURE": "0",                                # 保留你们的平台插件（必须只读）
        # 建议删除 NODE_TLS_REJECT_UNAUTHORIZED=0，改为注入 CA：
        # "NODE_EXTRA_CA_CERTS": "/opt/agent/ca/corp-ca.pem",
    }

    cmd = [
        "bwrap",
        "--die-with-parent", "--new-session",
        "--unshare-user", "--unshare-pid", "--unshare-ipc", "--unshare-uts", "--unshare-cgroup-try",
        # 若要独立网络：--unshare-net（但需额外方案让 LLM 网关可达，见 4.9）
        *RO_BINDS,                                     # /usr /lib /lib64 /bin /sbin /etc/ssl /etc/resolv.conf ...
        "--proc", "/proc", "--dev", "/dev",
        "--tmpfs", "/tmp", "--tmpfs", "/run",
        "--dir", "/home/tenant",
        "--ro-bind", str(template), "/opt/agent",      # 模板只读
        "--bind", str(tenant / "home"), "/home/tenant",
        "--bind", str(ws), "/workspace",               # ★ 唯一可写的业务目录
        "--chdir", "/workspace",                       # ★ cwd = 该用户该智能体的工作区
        "--setenv", "HOME", "/home/tenant",
        # ... 其余 setenv 与 env 一致
        "--",
        "/opt/agent/opencode", "serve",
        "--print-logs", "--log-level", "DEBUG",
        "--hostname", "127.0.0.1", "--port", str(port),
    ]

    log_fd = open(log_file, "a", encoding="utf-8", buffering=1)
    proc = subprocess.Popen(cmd, cwd=str(ws), start_new_session=True,
                            env=env, stdout=log_fd, stderr=subprocess.STDOUT)
    return proc
```

关键点（对应源码）：

- **`cwd` 必须是该用户的 `ws/`**：`serve` 默认以 `process.cwd()` 作为实例目录（`workspace-routing.ts:87`）。
- **`OPENCODE_DISABLE_PROJECT_CONFIG=1`**：否则用户的 `ws/opencode.json`、`ws/.opencode/plugin/*.ts` 会在进程内执行（`config.ts:420-424, 438-479`）。
- **`OPENCODE_CONFIG` + `OPENCODE_CONFIG_DIR` 指向只读模板**：保证模型、prompt、skill、插件集合由平台决定（`config.ts:415-418, 432-447`）。
- **`OPENCODE_DB` 独立**：会话物理隔离（`database.ts:44-54`）。
- **`OPENCODE_PERMISSION`**：控制面注入的规则集优先级在配置合并的最后（`config.ts:559-563`），用户改不动。
- **`OPENCODE_SERVER_PASSWORD` 每沙箱独立**：`server/auth.ts` 的 Basic Auth 生效；否则同 Pod 内任意进程都能直连别人的 127.0.0.1 端口。

### 4.3 沙箱技术选型（本地、可落地）

| 方案 | 特权要求 | 隔离强度 | 备注 |
|---|---|---|---|
| **bubblewrap（bwrap）** | 非特权（需允许 unprivileged userns）或 root | mount/pid/uts/ipc/cgroup ns + seccomp | **首选**：无守护进程、启动 ~10ms、用法即命令行 |
| **nsjail** | 非特权 userns 或 root | 同上 + cgroup/rlimit 原生支持 | 需要更细的 rlimit/cgroup 时用它 |
| **runc / youki / crun** | 需要 cgroup 写权限，通常需特权 Pod | 完整 OCI 容器 | 想要"真容器"且集群允许 `privileged` / cgroup delegation 时用 |
| **Landlock（内核 ≥5.13）** | **完全非特权** | 仅文件系统（+5.6.0 起可限制 TCP 端口） | 当 userns 被 seccomp/AppArmor 挡住时的保底方案，需自写 ~100 行 launcher |
| **gVisor（runsc）/ Kata** | 需节点级 RuntimeClass | 强（用户态内核/轻量 VM） | 走方案 C，交给 K8s |
| **Docker/containerd-in-Pod（DinD）** | 特权 + socket 挂载 | 强 | **不推荐**：把节点级权限带进业务 Pod |

K8s 侧的现实约束（务必先验证）：

1. **unprivileged user namespace**：容器内 `unshare(CLONE_NEWUSER)` 常被运行时默认 seccomp 策略拦截。可选做法：
   - 使用 **K8s ≥ 1.36**（用户命名空间已 GA，见 [Kubernetes v1.36：用户命名空间正式可用](https://kubernetes.io/zh-cn/blog/2026/04/23/userns-ga/)）并为 Pod 设置 `spec.hostUsers: false`；或
   - 在 1.30~1.35 上开启 `UserNamespacesSupport` 特性门控；
   - 或给容器加 `CAP_SYS_ADMIN` + 自定义 seccomp（放宽 `clone`/`unshare`）——**这是特权放宽，需安全评审**。
2. **AppArmor/SELinux** 也会拦住 userns，需要在节点上放行 profile。
3. **cgroup 配额**：K8s 默认不把 Pod 的 cgroup 子树委派给容器，`bwrap --unshare-cgroup` 能隔离视图但**限不了量**。单 Pod 内只能用 `prlimit`/`ulimit`（`--nproc`、`--nofile`、`--as`）做软限制；**CPU/内存硬配额必须靠方案 C 的每租户 Pod**。

bwrap 启动前的自检（建议做成 Pod 启动探针）：

```bash
unshare --user --pid --mount-proc true 2>/dev/null && echo "userns OK" || echo "userns BLOCKED"
bwrap --ro-bind / / -- true && echo "bwrap OK"
test -w /sys/fs/cgroup && echo "cgroup delegated" || echo "cgroup read-only (只能 ulimit)"
```

**补充（应急降级）：把 `config.shell` 指向自写 wrapper**。V1 的 bash 工具通过 `Shell.acceptable(cfg.shell)`（`packages/opencode/src/tool/shell.ts:600`）解析 shell 路径，只拦 `fish`/`nu`（`packages/core/src/shell.ts:13-23`），绝对路径的可执行文件一律接受；POSIX 下 bash 工具的调用形态是 `wrapper -c "<command>"`（`shell.ts:303-309`）；而 `!command` 走 `Shell.args`，bash 分支会传 `-l -c '<script>' opencode <cwd>`（`packages/core/src/shell.ts:183-196`、调用点 `session/prompt.ts:523-524`）。所以可以写一个 wrapper 读环境变量里的沙箱标识，再 `exec bwrap ... /bin/bash "$@"`。

**但必须清楚它的边界**：这只包住了 bash 工具，**包不住** PTY、`!command`、插件 `Bun.$`、MCP、LSP（§1.2(10)）。因此它只能是"阶段 P2 之前"的过渡措施，不能替代进程级沙箱。

### 4.4 会话隔离

- 首选：**每租户一个 `opencode.db`**（`OPENCODE_DB`）。
- 若希望"换智能体也不串会话"，可进一步用 `(tenant, agent)` 一库：`.../agents/<agent>/db/opencode.db`。代价是连接数变多、备份碎片化，建议先按租户。
- **不要**指望 `project_id` 做隔离（`git remote`/root commit 会碰撞，非 git 全落 `global`）。
- 若必须让同一用户跨设备续聊：把 `session_id → tenant` 写进控制面数据库，前端只持 opaque 的 `conversation_id`。
- 会话备份/迁移：停机后 copy `opencode.db` + `-wal`/`-shm`，或 `sqlite3 .backup`。

### 4.5 智能体模板挂载

- `mgr` 里 `agent_template` 物化为 `/opt/agents/<agent>/`，由"配置管理服务"负责内容与版本；沙箱内只读。
- **技能**：模板配置里用绝对路径指向只读目录：

```jsonc
// /opt/agents/<agent>/opencode.json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": { "paths": ["/opt/agent/skills"] },   // 对应 packages/skills PVC（ro）
  "permission": { "external_directory": "deny" }
}
```

  （`skills.paths` 支持绝对路径，见 `packages/opencode/src/skill/index.ts:210-220`。）
- **插件**：`/opt/agent/config/plugin/*.ts`（对应 `.opencode/plugin`）必须由平台提供且只读；插件在进程内拥有完全权限，等同"平台代码"。
- **热更新**：模板变更后不要原地改，改为"新模板目录 + 滚动重建沙箱"，避免半旧半新。

### 4.6 权限与配置收敛（纵深防御）

由控制面注入（`OPENCODE_PERMISSION`，用户不可改）：

```json
{
  "external_directory": "deny",
  "bash": "ask",
  "webfetch": "deny",
  "websearch": "deny",
  "task": "ask",
  "read": { "*": "allow", "*.env": "deny" },
  "edit": "allow"
}
```

配套要求：

1. `OPENCODE_DISABLE_PROJECT_CONFIG=1`（否则规则会被用户项目配置覆盖/注入）。
2. `bash: "ask"` 时不接人工，由控制面**策略引擎代答**：订阅 `permission.asked` 事件（发布点 `packages/opencode/src/permission/index.ts:96-107`，schema `packages/schema/src/v1/permission.ts:27-66`），按"命令前缀白名单 + 路径必须在 `/workspace`"批准，其余拒绝，再 `POST /session/:id/permissions/:permissionID` 回复（`groups/session.ts:101`、`handlers/session.ts:362-367`）。
   ⚠️ **不要用插件钩子 `permission.ask` 做审批网关**：它只在 `packages/plugin/src/index.ts:261` 有类型声明，**全仓库没有任何 `plugin.trigger("permission.ask", ...)` 调用，是死代码**。插件侧唯一能"拒绝工具"的手段是在 `tool.execute.before` 抛异常（`packages/opencode/src/plugin/index.ts:284-297`：顺序 await、就地 mutate output，异常变成 defect 使该次工具调用失败）。
3. `external_directory: "deny"` 会让越界访问**直接报错**而不是等待确认（`tool/external-directory.ts:35-40`）。这是 V1 最小代价的"边界配置"，但**不是内核级 jail**（绝对路径、`../`、symlink 在 ask/allow 下都会放行，见 §9）。
4. **收敛所有绕过权限的执行入口**（§1.2(10)）：PTY（`/pty*`、`/pty/:id/connect`）、`/session/:id/shell`（`groups/session.ts:98`）、`!command`（`session/prompt.ts:552-578`）、MCP server 配置、插件加载。webchat 不需要终端就直接在网关层拒掉这些路径。
5. 沙箱内不放 `~/.git-credentials`、`~/.aws`、`kubeconfig`、SA token（`automountServiceAccountToken: false`）。
6. **不要**使用 `--auto`/`--yolo`/`--dangerously-skip-permissions` 类全放开开关（`cli/cmd/run.ts:242-274, 801-821`）。
7. `NODE_TLS_REJECT_UNAUTHORIZED=0` 必须去掉，改为注入内网 CA（`NODE_EXTRA_CA_CERTS`）。
8. 不要把 `/global/*` 暴露给终端用户：`POST /global/dispose` 会 dispose **所有**实例（`groups/global.ts:117-125`），`GET /global/event` 是跨目录广播。

### 4.7 生命周期、池化与资源

**状态机**：

```
absent ──ensure()──▶ starting ──健康检查(/config 或 /session 200 + 口令)──▶ ready
   ▲                    │                                                   │
   │                    └──失败 ──▶ failed（退避重启，最多 N 次）              │
   └────────destroy()────────── draining ◀──stop()/idle 超时(默认15min)──────┘
```

**要点**：

- **键**：`(tenant_id, agent_name)`；`ensure()` 加 per-key 锁，避免并发重复起进程。
- **端口**：从 `conf/BASE_PORT` 起的区间分配（如 17000–18999），alloc/free 记在控制面；端口冲突自动重试。
- **健康检查**：`GET /config`（带 Basic Auth）返回 200 且 `directory` 正确。
- **空闲回收**：无活动会话 + 无 inflight 请求超过 TTL → `stop()`；`OPENCODE_DB` 保留数据。
- **进程组**：`start_new_session=True` + `bwrap --die-with-parent`；销毁时按 **pgid** kill（参考 opencode 自己的 `killTree`：`packages/core/src/shell.ts:31-60`）。
- **日志**：`logs/<tenant>/<agent>/`，按大小轮转（防止用户把 Pod 磁盘刷满）。
- **配额**：`prlimit --nproc=256 --nofile=4096 --as=4G`；磁盘用项目配额（XFS project quota）或独立 PVC + 容量上限。
- **回收度量**：记录每沙箱 RSS/进程数，超阈值告警（Bun 进程常驻 100–250MB，别把 Pod 打爆）。
- **规模估算**：`N_users × N_agents` 个进程是上限；若 500 用户 × 5 智能体 = 2500 进程不可行 ⇒ 采用
  "**每用户一个进程 + 多目录**"合并策略（一个进程内 `x-opencode-directory` 指向该用户的多个 agent 工作区），详见 §4.8。

### 4.8 规模化：每用户一进程 + 多目录合并策略（可行性、边界、源码改动点）

**结论：可以做，而且不改 opencode 源码也能做**——但必须遵守一条纪律：**所有参与"配置/插件发现"的路径都必须是只读挂载（或根本不存在）**。做不到这条纪律时，才需要改源码（改动点见 §4.8.4）。

#### 4.8.1 依据：一个进程内哪些是"按目录隔离"、哪些是"进程级"

| 维度 | 作用域 | 证据 |
|---|---|---|
| **Config（含 agent 定义、prompt、permission、skills.paths、插件列表、default_agent、model）** | **✅ 按目录** | `packages/opencode/src/config/config.ts:614-616`：`InstanceState.make(... loadInstanceState(ctx))`，配置从 `ctx.directory` 向上解析 |
| Agent / Skill / Command / Plugin hooks / Permission / ToolRegistry / LSP / Watcher / MCP / Formatter / Snapshot / Provider / VCS / Question / BackgroundJob | **✅ 按目录** | `InstanceState.make` 共 23 处，覆盖 `agent/agent.ts:98`、`skill/index.ts:259,273`、`command/index.ts:159`、`plugin/index.ts:134`、`permission/index.ts:46`、`tool/registry.ts:121`、`lsp/lsp.ts:145`、`mcp/index.ts:492`、`format/index.ts:38`、`snapshot/index.ts:66`、`provider/provider.ts:1451` 等 |
| **每智能体主提示词** | **✅ 按目录** | 配置字段 `agent.<name>.prompt`（`packages/core/src/v1/config/agent.ts:20`、应用点 `packages/opencode/src/agent/agent.ts:283`），最终作为 system prompt（`packages/opencode/src/session/llm/request.ts:60`：`input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(model)`）。且支持 `{file:...}` 从文件读取（`packages/opencode/src/config/variable.ts:33-90`，相对路径按声明它的配置文件目录解析） |
| 每个 agent 的 `.opencode/plugin/*.ts` | ✅ 按目录加载 | `packages/opencode/src/config/config.ts:476-479` 对每个 config 目录调用 `ConfigPlugin.load(dir)` |
| **`OPENCODE_DB`（会话库）** | ❌ **进程级** | `packages/core/src/database/database.ts:43-55` + `makeGlobalNode`（`:57`）：一个进程一个库 |
| **`OPENCODE_PERMISSION`（env 规则集）** | ❌ 进程级，且**最后合并、覆盖同键** | `packages/opencode/src/config/config.ts:559-565`（`mergeDeep(result.permission ?? {}, JSON.parse(Flag.OPENCODE_PERMISSION))`） |
| `OPENCODE_CONFIG` / `OPENCODE_CONFIG_DIR` / `OPENCODE_PURE` / `OPENCODE_DISABLE_*` / `OPENCODE_WORKSPACE_ID` | ❌ 进程级 env | `packages/core/src/flag/flag.ts`、`packages/core/src/global.ts:59-72` |
| 插件 JS 模块缓存 | ❌ 进程级（ES module cache） | `packages/opencode/src/plugin/loader.ts:139` `await import(row.entry)` —— 同路径只 import 一次，模块级单例状态在同一用户的 agent 间共享 |
| Instance 缓存 | ⚠️ 进程级 Map，**无 TTL/容量上限** | `packages/opencode/src/project/instance-store.ts:43`（只在 dispose/finalizer 清理） |
| Session 执行调度 | 进程级协调器、**按 session 串行、不同 session 并行** | `packages/core/src/session/run-coordinator.ts:5-15` |

**结论**：per-agent 的"提示词 / 配置 / 插件 / skills / 模型 / 权限细则"全都能按目录区分；不能按目录区分的只有 5 样：**会话库、SERVER_PASSWORD、OPENCODE_PERMISSION、全局 disable 开关、插件模块缓存**。而前两样恰好是我们**希望**按用户统一的东西。

#### 4.8.2 零改动的落地形态：沙箱内挂载布局

```
/var/sandbox/tenants/<uid>/                 # 一个用户 = 一个 bwrap = 一个 opencode serve = 一个 DB
├── home/                                   # rw：XDG data/cache/state（会话库、日志、snapshot）
├── config/                                 # ro：平台下发的"用户级全局配置"（可选，通常留空）
└── agents/<agent>/
    ├── tpl/                                # ro：来自 /opt/agents/<agent> 模板
    │   ├── opencode.json                   #    内含 agent.<name>.prompt = {file:./prompt/session/default.txt}
    │   ├── opencode.jsonc                  #    ★ 两个名字都要占住，见下
    │   ├── .opencode/{agent,command,plugin}/
    │   ├── prompt/…  skills/…
    │   └── workspace/                      #    空目录，作为挂载点
    └── ws/                                 # rw：该用户该智能体的真实工作区
```

沙箱内视角与 bwrap 关键片段：

```bash
# 目录视图
/agents/<agent>/            ← ro-bind  <tenant>/agents/<agent>/tpl
/agents/<agent>/workspace/  ← rw-bind  <tenant>/agents/<agent>/ws     # ★ x-opencode-directory 指向它
/home/tenant/               ← rw-bind  <tenant>/home
/home/tenant/.config/opencode/  ← ro-bind <tenant>/config             # 全局配置目录只读

# bwrap（在 §4.2 的基础上增补）
--ro-bind "$TPL" /agents/"$AGENT" \
--bind   "$USER_WS" /agents/"$AGENT"/workspace \
--ro-bind "$TPL/opencode.json"  /agents/"$AGENT"/workspace/opencode.json \
--ro-bind "$TPL/opencode.jsonc" /agents/"$AGENT"/workspace/opencode.jsonc \
--ro-bind "$TPL/.opencode"      /agents/"$AGENT"/workspace/.opencode \
--bind   "$TENANT_HOME" /home/tenant \
--ro-bind "$TENANT_CONFIG" /home/tenant/.config/opencode \
--chdir /agents/"$AGENT"/workspace
```

为什么必须"把配置再挂进 workspace 一层"：配置发现是**从实例目录 D 向上走到 worktree**（`ConfigPaths.files("opencode", ctx.directory, ctx.worktree)`，`packages/opencode/src/config/paths.ts:10-21`），而 `up()` **包含起始目录自身**（`packages/core/src/fs-util.ts:168-182`）。如果用户的 workspace 本身是个 git 仓库，`worktree === D`，向上走一步就停 —— 父目录的模板配置根本不会被读到。把 `opencode.json`/`opencode.jsonc`/`.opencode` 只读挂到 D 上，就与"是不是 git 仓库"无关了。

**唯一纪律（三次重复都不为过）**：参与配置发现的**每一个**可写路径都会被当成 injection 面。必须只读或不存在的有：

| 路径 | 谁在找它 | 不设防的后果 |
|---|---|---|
| `D/opencode.json`、`D/opencode.jsonc`、`D/.opencode/` | `config/paths.ts:10-41` | 用户写 `opencode.jsonc`（**注意 jsonc 在 `toReversed()` 后是后合并者，会覆盖 json**）即可注入 plugin/permission/model |
| `<D 的各级祖先>/opencode.json(c)`、`.opencode/` | 同上（非 git 时一路走到 `/`） | 同上 |
| `$HOME/.opencode/` | `config/paths.ts:30` | 同上（HOME 若可写） |
| `$XDG_CONFIG_HOME/opencode/`（=`Global.Path.config`） | `config/paths.ts:26`、`config.ts:413` | 同上 + 插件 node_modules 安装点 |
| `OPENCODE_CONFIG` 指向的文件 | `config.ts:415-418` | 同上 |
| `OPENCODE_CONFIG_DIR` 指向的目录 | `config/paths.ts:39` | 同上 |

**用户个性化需求怎么满足**：不要让用户直接写配置文件，而是由平台把"用户级覆盖"写入 `tenants/<uid>/config/`（或每个 agent 的 overlay），**以只读方式挂载**。这样既有个性化，又没有注入面。

**每请求仍必须带目录**：`x-opencode-directory: /agents/<agent>/workspace`（或 `?directory=`），并且**永远显式传**，因为：

- `GET /session` 不带 `?directory=` 时只按 project 过滤（`handlers/session.ts:64-75`：`ctx.query.directory ? InstanceState.directory : undefined`），而同一用户的多个 agent 很可能是同一个仓库的多个 clone ⇒ **同一个 project id**（`packages/core/src/project.ts:73-122`），会话列表会跨 agent 串味。
- ⚠️ **只在请求头里带 `x-opencode-directory` 是不够的**：上面的过滤条件是 `ctx.query.directory`（**URL 查询参数**）。列表类接口要同时带 `?directory=<同一个值>`，否则实例对了、过滤没生效。建议控制面统一给所有出站请求同时加头与查询参数（SDK 的做法也是如此：头始终设，GET/HEAD 额外补查询参数，见 `packages/sdk/js/src/v2/client.ts:18-48,63-68`）。
- 需要释放某 agent 的资源时，用 `POST /instance/dispose?directory=...`（`groups/instance.ts:44,62-71`、`handlers/instance.ts:24-27`）——**每目录粒度的回收，零改动可用**。

#### 4.8.3 必须接受的代价（前 5 条是核心，后 5 条是合并后才会暴露的细节）

1. **一个用户一个会话库**：`OPENCODE_DB` 是进程级，无法按 agent 分库。"会话不跨 agent 串"靠 `directory` 列 + 控制面强制传参保证（控制面还必须做 session 归属校验，见 §5.1(2)）。
2. **权限规则集是进程级的**：`OPENCODE_PERMISSION` 最后合并并覆盖同键 ⇒ 把它当成**安全下限**（`external_directory: "deny"` 等），per-agent 的差异写在各自 `opencode.json` 的非冲突键上。
3. **崩溃/插件缺陷的爆炸半径 = 整个用户**：一个 agent 的插件把进程搞挂，该用户所有 agent 一起挂。对加载第三方插件的 agent，建议仍单独开进程。
4. **插件模块级单例状态在同一用户的 agent 间共享**（ES module cache）；不要在同一进程内混用来源不同的插件。
5. **一个事件循环**：5 路同时流式没问题（I/O 密集），但重 CPU 的插件逻辑会互相影响。

还有几条"合并后才会暴露"的细节（审计补充）：

6. **配置/模板变更不是即时生效的**：目录的 config 缓存在 `InstanceState`（`config/config.ts:614-618`），v2 侧整套 location 服务还有 60 分钟 idle TTL（`packages/core/src/location-services.ts:109`）。改完模板要 `POST /instance/dispose?directory=...` 主动失效，否则要等 TTL。
7. **非 git 目录会让 project 塌缩成 `"global"`**（`packages/core/src/project.ts:112`），由此产生三个同用户内的串扰：
   - 会话列表：不传 `?directory=` 时就是"该用户所有非 git 工作目录"的集合；
   - v2 的 PermissionSaved 规则按 `project_id` 落库（`packages/core/src/permission/saved.ts:46`）⇒ 非 git 目录之间共享 "always allow" 记忆（v1 的 `approved` 是目录内存态，无此问题）；
   - snapshot 影子库路径 = `<data>/snapshot/<project.id>/<hash(worktree)>`（`snapshot/index.ts:71`），非 git 时 `worktree="/"`（`project/project.ts:217`）⇒ 多个 agent 目录可能指向同一个影子库。
   **规避办法（零改动）**：给每个 agent 的工作目录一个 git 边界（`git init` 会产生不同的 root commit ⇒ 不同 project id）；或接受并靠"永远显式传 directory + v1 权限模型"。
8. **全局 `<Global.Path.config>/AGENTS.md` 会无条件进入每个目录的系统提示**（`session/instruction.ts:60-63,115-120`）——它落在用户的 config 目录里，所以要么只读挂载、要么别放敏感内容。
9. **MCP OAuth 回调是进程级单例**（固定端口/路径，`packages/opencode/src/mcp/oauth-callback.ts:9-22`）：同一进程内多个 agent 同时走 OAuth 型 MCP 会冲突。
10. **v1 的 runner 是按目录的**（`session/run-state.ts:35-49`）：同一个 session 被两个不同目录的请求驱动，可能绕过 BusyError 起两个 runner。控制面把 session 钉死到唯一目录即可避免。

#### 4.8.4 真要改源码的话：最小改动点

**最高性价比的一条**（建议即使不改其它也加上）：在 `requireSession` 里断言会话目录等于当前实例目录 —— `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts:81-87`（或 `session/session.ts:540-544` 的 `get`）。虽然 §1.2(5) 已说明 middleware 会按 session 目录绑定实例、外部改 header 无法重定向，但**在合并进程里这是"用户内 agent 互串"的最后一道保险**，成本约 3 行。

其余改动点（只有当你**不愿意**依赖"只读挂载纪律"、想在进程内做"可信配置根"时才需要）：

| # | 文件与函数 | 改动 | 量级 |
|---|---|---|---|
| 1 | `packages/opencode/src/config/paths.ts`：`files()`(:10-21)、`directories()`(:23-41) | 增加"可信根"过滤：新 flag `OPENCODE_CONFIG_ROOTS`（逗号分隔的绝对路径）；只保留位于这些根之下（或恰为这些根）的配置源；非可信根的项目配置直接跳过 | ~30 行 |
| 2 | `packages/core/src/flag/flag.ts` | 注册 `OPENCODE_CONFIG_ROOTS`（getter 形式，运行时求值） | ~3 行 |
| 3 | `packages/opencode/src/config/config.ts`：`loadInstanceState`(:328-490) | 用同一个白名单过滤 `ConfigPaths.files(...)` 与 `.opencode` 目录的合并；`pluginScopeForSource`(:337-342) 已有"是否在实例内"的判定，可直接复用 | ~15 行 |
| 4 | `packages/opencode/src/project/instance-store.ts` | 给实例 `Map`(:43) 加容量上限或 TTL（单用户 agent 数很多时才需要）；否则用 `POST /instance/dispose` 手动回收 | ~20 行（可选） |
| 5 | `packages/opencode/src/config/config.ts:559-565` | 若希望 `OPENCODE_PERMISSION` 作为"默认值"而不是"最终覆盖"，改成先合并 config 再合并 env 的反向顺序（当前行为对安全其实更有利，**建议不改**） | ~5 行（可选） |

**不建议改的**：把 `Database` 从全局 node 改成按目录（`packages/core/src/database/database.ts:57`）——它被几乎所有服务引用，改动面极大，收益（per-agent 分库）可以通过"控制面强制传 directory + 归属校验"替代。

**一个用户一个进程 vs 一个 (用户,智能体) 一个进程，怎么选**：

| | 每 (用户,智能体) 一进程 | 每用户一进程 + 多目录 |
|---|---|---|
| 进程数（500 用户 × 5 智能体，全部活跃） | 2500 | 500 |
| 内存（按 Bun 常驻 100–250MB 估） | 250–600GB ❌ | 50–125GB ⚠️（仍偏高，需要池化） |
| 会话库 | 每实例独立（天然隔离） | 每用户一个（靠 directory + 控制面校验） |
| 权限/配置粒度 | 完全独立 | 按目录，env 类只能统一 |
| 崩溃半径 | 单智能体 | 整个用户 |
| 配置注入面 | 小（进程 env 即可） | **必须靠只读挂载纪律** |

**推荐**：`活跃并发 × 智能体数` 才是真实进程数上限。先做**空闲回收**（§4.7 的 TTL）+ **每 (用户,智能体) 一进程**，把并发压到可控范围；当单 Pod 进程数逼近上限（经验值 200–400 个）时，再对"长尾用户"启用合并策略（同一个用户的多智能体共享一个进程），并用"混合模式"保留重资源 agent 的独立进程。

### 4.9 网络与凭据

- **LLM 凭据不进沙箱**：控制面网关持有真实 key；沙箱拿到的是"该租户的一次性/短期 token + 内网网关地址"。这样即使沙箱被攻破，key 不泄漏、可撤销、可限流。
- **出网白名单**：单 Pod 内只能做应用层（`HTTP_PROXY`/`HTTPS_PROXY`）与 DNS 层面的软约束；**硬隔离放到方案 C 的 `NetworkPolicy`/Cilium FQDN**。
- 若必须在单 Pod 内做强网隔离：`--unshare-net` + 宿主侧代理，通过 unix socket 传入沙箱（bwrap `--bind /run/egress/<tenant>.sock`），沙箱内只有 loopback 与这个 socket。**前提是 opencode 的 HTTP 客户端遵循代理环境变量，需要实测**（Bun 的 `fetch` 对 `HTTPS_PROXY` 支持、或 Node 侧 `NODE_USE_ENV_PROXY=1`）；不成立时退化为方案 C。
- 每沙箱 `OPENCODE_SERVER_PASSWORD` 独立；`serve` 只监听 `127.0.0.1`（默认 `hostname=127.0.0.1`，见 `cli/network.ts:12-16`），**绝不要 `--mdns` 或 `--hostname 0.0.0.0`**。

### 4.10 具体布局示例（方案二：每 (租户, 智能体) 一进程 + bwrap）

适用前提（本方案的典型业务形态）：**智能体不是 CodeAgent，但运行中会自己写脚本并执行**。因此：

- 工作区不是 git 仓库 ⇒ `project_id = "global"`、`vcs = undefined`；**snapshot 完全不会启用**（`packages/opencode/src/snapshot/index.ts:168-169`：`if (state.vcs !== "git") return false`），无需担心影子库。
- 工作区边界 = 实例目录本身（`packages/opencode/src/project/instance-context.ts:20-22`：非 git 时跳过 worktree 检查）⇒ `external_directory` 的判定范围恰好就是 `/workspace`。
- 进程级 env 就够用 ⇒ **可以打开 `OPENCODE_DISABLE_PROJECT_CONFIG=1`**，让用户工作区彻底不参与配置/插件发现（§4.8 那套"只读挂载纪律"在方案二里不需要）。

#### 4.10.1 三个视角对照（示例租户 `t_1024`，智能体 `fengkong`，端口 17001）

**视角 A：宿主磁盘（`df`/`ls` 看到的）**

```
/var/sandbox/                                   root:root      0755
└── tenants/
    └── t_1024/                                 root:root      0700   ← 只有 mgr(root) 能进
        ├── meta.json                           root:root      0600   租户/配额/创建时间
        ├── logs/                               root:root      0755
        │   └── fengkong/
        │       └── opencode-17001.log          root:root      0644   宿主侧 stdout 抓取
        └── agents/
            └── fengkong/
                ├── home/                       u1024-fk:…     0700   ★ 该 (租户,智能体) 的 HOME
                │   ├── .config/opencode/       rw                    全局配置目录（通常为空）
                │   ├── .local/share/opencode/  rw
                │   │   ├── opencode.db         + -wal / -shm  ★ 会话库
                │   │   ├── log/opencode.log    rw                    opencode 自带日志
                │   │   ├── tool-output/        rw                    超长工具输出落盘
                │   │   ├── repos/  storage/    rw
                │   ├── .local/state/opencode/  rw                    plugin-meta.json, locks/
                │   └── .cache/opencode/        rw                    models.json, bin/rg(未预装时)
                ├── ws/                         u1024-fk:…     0750   ★ 用户工作区
                │   ├── scripts/report.py              智能体自己写的脚本
                │   ├── data/input.xlsx                用户上传的输入
                │   ├── output/result.csv              脚本产出
                │   └── .tmp/                          TMPDIR 指向这里（见 4.10.4）
                └── run/                        root:root      0755   ★ 宿主侧状态，不进沙箱
                    ├── pid   port   sandbox.json(root:root 0600)
                    └── token                                0600   该沙箱的 SERVER_PASSWORD

/opt/agents/                                    root:root      0755   ← 所有租户共享的只读模板
└── fengkong/
    ├── opencode                                root:root      0755   190MB，多租户共用同一 inode
    ├── opencode.json                           root:root      0644   OPENCODE_CONFIG
    ├── .gitignore                              root:root      0644   ★ 必需，见 4.10.5
    ├── config/                                 root:root      0755   OPENCODE_CONFIG_DIR
    │   ├── .gitignore                          root:root      0644   ★ 必需
    │   ├── node_modules/@opencode-ai/plugin/   root:root      0755   预装，避免启动时拉包
    │   ├── agent/  command/  plugin/*.ts       root:root      0644   平台插件（file 源）
    ├── prompt/
    │   ├── session/default.txt                 root:root      0644   主提示词（{file:} 引用）
    │   └── agent/  tool/                                        内置子智能体/工具提示词
    └── skills/                                 root:root      0755   构建期从 PVC 物化（或运行期单独挂载，见视角 B）
```

> ⚠️ 不要用 `symlink` 指向 `/packages/skills/...`：符号链接的目标路径必须也在沙箱的挂载表里，否则一切（沙箱内）解析失败。要么**构建期物化**到模板目录，要么**单独 ro-bind**（视角 B 的最后一行）。

**视角 B：挂载表（`bwrap` 参数 → 沙箱内路径）**

| 宿主 | 沙箱内 | 模式 | 为什么 |
|---|---|---|---|
| `/usr` `/lib` `/lib64` `/bin` `/sbin` | 同名 | ro | 提供 `bash`/`python3`/`node`/`rg`，智能体写的脚本要能跑 |
| `/etc/ssl` `/etc/ca-certificates` `/etc/resolv.conf` `/etc/hosts` `/etc/nsswitch.conf` `/etc/passwd` `/etc/group` `/etc/ld.so.cache` `/etc/localtime` | 同名 | ro | TLS、DNS、uid 名称解析；**不要整个 `/etc` 挂进去** |
| `/opt/agents/fengkong` | `/opt/agent` | **ro** | 二进制 + 配置 + 提示词，平台资产 |
| `/packages/skills/fengkong`（PVC） | `/opt/agent/skills` | **ro** | 技能库（若未在构建期物化进模板则单独挂） |
| `…/agents/fengkong/home` | `/home/tenant` | **rw** | `HOME`，XDG 四件套都在这下面 |
| `…/agents/fengkong/ws` | `/workspace` | **rw** | ★ 唯一业务读写区，`cwd` |
| tmpfs | `/tmp` `/run` | rw | 沙箱私有临时区，重启即消失 |
| procfs / devfs | `/proc` `/dev` | — | `--proc` / `--dev` |
| （不挂） | — | — | `…/run`、`/var/sandbox/tenants` 根、其他租户目录一律**不出现**在沙箱里 |

关键点：**`ws` 与 `home` 是平级的两个挂载点，`ws` 不在 `home` 里面**。这样 `$HOME/.opencode`、`$HOME/.config` 的向上查找永远走不到用户工作区，反之清空 HOME 也不会动用户产出。

**视角 C：沙箱内看到的目录树 + 环境变量 + 启动命令**

```
沙箱内（bwrap 之后）：
/home/tenant/{.config,.local,.cache}/…    ← HOME（rw）
/workspace/                               ← cwd，也是 x-opencode-directory（rw）
/opt/agent/{opencode,opencode.json,config/,prompt/,skills/}   ← 只读模板（skills 来自 PVC 的 ro-bind）
/usr/bin/{bash,python3,rg,node} … /lib …  ← 只读系统
/tmp /run                                 ← tmpfs
```

```bash
# 环境变量（Popen env，沙箱内视角）
HOME=/home/tenant
XDG_CONFIG_HOME=/home/tenant/.config
XDG_DATA_HOME=/home/tenant/.local/share
XDG_STATE_HOME=/home/tenant/.local/state
XDG_CACHE_HOME=/home/tenant/.cache
TMPDIR=/workspace/.tmp                       # ★ 见 4.10.4

OPENCODE_CONFIG=/opt/agent/opencode.json      # 只读
OPENCODE_CONFIG_DIR=/opt/agent/config         # 只读（= Global.Path.config）
OPENCODE_DISABLE_PROJECT_CONFIG=1             # ★ 方案二的关键：ws 不参与配置/插件发现
OPENCODE_PERMISSION='{"external_directory":{"*":"deny","/workspace/*":"allow","/usr/*":"allow","/opt/agent/*":"allow","/home/tenant/*":"allow"},"webfetch":"deny","websearch":"deny"}'
OPENCODE_SERVER_PASSWORD=<每沙箱随机 32B>
OPENCODE_DISABLE_MODELS_FETCH=1
OPENCODE_DISABLE_AUTOUPDATE=1
OPENCODE_DISABLE_LSP_DOWNLOAD=1
OPENCODE_DISABLE_EMBEDDED_WEB_UI=1
# 脚本类智能体需要的语言运行时缓存（都落在 HOME 里，可写、按沙箱隔离）
PYTHONUSERBASE=/home/tenant/.local
PIP_CACHE_DIR=/home/tenant/.cache/pip
PIP_DISABLE_PIP_VERSION_CHECK=1
NPM_CONFIG_CACHE=/home/tenant/.cache/npm
```

```bash
# bwrap（POSIX路径用 -- 分隔）
bwrap \
  --die-with-parent --new-session \
  --unshare-user --unshare-pid --unshare-ipc --unshare-uts --unshare-cgroup-try \
  --cap-drop ALL \
  --ro-bind /usr /usr --ro-bind /lib /lib --ro-bind /lib64 /lib64 \
  --ro-bind /bin /bin --ro-bind /sbin /sbin \
  --ro-bind /etc/ssl /etc/ssl --ro-bind /etc/ca-certificates /etc/ca-certificates \
  --ro-bind /etc/resolv.conf /etc/resolv.conf --ro-bind /etc/hosts /etc/hosts \
  --ro-bind /etc/nsswitch.conf /etc/nsswitch.conf \
  --ro-bind /etc/passwd /etc/passwd --ro-bind /etc/group /etc/group \
  --ro-bind /etc/ld.so.cache /etc/ld.so.cache --ro-bind /etc/localtime /etc/localtime \
  --proc /proc --dev /dev --tmpfs /tmp --tmpfs /run \
  --ro-bind /opt/agents/fengkong /opt/agent \
  --ro-bind /packages/skills/fengkong /opt/agent/skills \
  --bind   /var/sandbox/tenants/t_1024/agents/fengkong/home /home/tenant \
  --bind   /var/sandbox/tenants/t_1024/agents/fengkong/ws   /workspace \
  --chdir /workspace \
  --setenv HOME /home/tenant \
  --setenv OPENCODE_CONFIG /opt/agent/opencode.json \
  --setenv OPENCODE_CONFIG_DIR /opt/agent/config \
  --setenv OPENCODE_DISABLE_PROJECT_CONFIG 1 \
  --setenv OPENCODE_SERVER_PASSWORD "$TOKEN" \
  -- /opt/agent/opencode serve --print-logs --log-level DEBUG \
       --hostname 127.0.0.1 --port 17001
```

#### 4.10.2 一次对话的落点（脚本类智能体）

| 步骤 | 发生的事 | 落到哪个路径（宿主视角） |
|---|---|---|
| 1 | 用户提问，会话创建 | `…/fengkong/home/.local/share/opencode/opencode.db` |
| 2 | 智能体 `write` 工具写脚本 | `…/fengkong/ws/scripts/report.py`（沙箱内 `/workspace/scripts/report.py`） |
| 3 | 智能体 `bash` 执行 `python3 scripts/report.py` | 进程在沙箱内，cwd=`/workspace`；产出 `ws/output/result.csv` |
| 4 | 脚本缺依赖，`pip install --user openpyxl` | 包落 `…/home/.local/lib/python3.x/site-packages/`，缓存落 `…/home/.cache/pip/` |
| 5 | 工具输出超 50KB（默认 `maxBytes=51200`） | 全文落 `…/home/.local/share/opencode/tool-output/`（`tool/truncation-dir.ts:4`） |
| 6 | 运行日志 | 宿主 `…/t_1024/logs/fengkong/opencode-17001.log` + 沙箱内 `…/home/.local/share/opencode/log/opencode.log` |
| 7 | 沙箱重启 | `home/`、`ws/` 全部保留（会话续聊、产出物在）；`/tmp`、`/run` 清空 |

第 4 步是这类业务的关键：**依赖要装到 HOME 里，而不是镜像里**（镜像只读、多租户共享）。若集群不出网，改为在镜像里预置常用包，并设 `PIP_NO_INDEX=1`/`PIP_FIND_LINKS=/opt/wheels`。

#### 4.10.3 需要预置或预装的清单（否则运行时会踩坑）

| 项 | 原因（源码证据） | 做法 |
|---|---|---|
| 每个只读配置目录里放 `.gitignore` | `ensureGitignore` 会 `ensureDir(dir)` 再写 `.gitignore`，调用处是 `Effect.orDie`（`config/config.ts:309-326,450`）；只读文件系统上写入是否被 `catchIf(PermissionDenied)` 兜住取决于 EROFS 的错误映射，**预置文件则根本不走写入分支** | 模板里放 `.gitignore`（内容：`node_modules/package.json/package-lock.json/bun.lock/.gitignore`） |
| 配置目录里预装 `node_modules/@opencode-ai/plugin` | `Npm.install` 先探测可写性，不可写直接返回（`core/npm.ts:148-152`）；`node_modules` 已存在也会短路（`:157-165`） | 在模板 `config/` 下预装，避免启动期网络依赖 |
| 平台插件用**本地 file 源** | npm 源插件会走 `Npm.add` 写到 `$XDG_CACHE_HOME/opencode/packages/` 并访问 registry（`core/npm.ts:123-145`） | `"plugin": ["file:///opt/agent/config/plugin/xxx.ts"]`（file 源插件需 `export id`） |
| 镜像里装 `ripgrep` 并确保在 PATH | `grep` 工具先 `which("rg")`，否则下载到 `$XDG_CACHE_HOME/opencode/bin/rg`（`core/ripgrep/binary.ts:94-120`） | 基础镜像 `apt install ripgrep`（只读 `/usr` 即可用），沙箱内零下载 |
| 镜像里装 `python3` / `node` / `bash` 等运行时 | 智能体会自己写脚本并执行 | ro-bind `/usr` 已覆盖；确认所需解释器与常用系统库都在 |
| 主提示词写明"产出写到 `/workspace`，不要用 `/tmp`" | `/tmp` 是 tmpfs，重启即丢；且 `external_directory` 若设为 deny，写 `/tmp` 会被拒 | 在 `prompt/session/default.txt` 里固化约定 |

#### 4.10.4 `TMPDIR` 放哪里：一个容易被忽略的取舍

- 默认 `TMPDIR=/tmp`（tmpfs）：干净、重启即清；但**它在 `/workspace` 之外**——如果 `external_directory` 设了 `deny`，智能体用文件工具写 `/tmp/x.py` 会被直接拒绝；`opencode` 自己的截断目录也变成 `/tmp/opencode`（重启丢失）。
- 推荐 `TMPDIR=/workspace/.tmp`：临时文件落在业务区内，**不触发 external_directory**，重启可见（便于排障），也能被配额统计到。代价是需要在 workspace 里预留/忽略该目录。
- 两者可并存：`/tmp` 仍挂 tmpfs（给硬编码 `/tmp` 的程序用），`TMPDIR` 指向 workspace。

#### 4.10.5 UID 与权限的现实约束（先验证再动手）

bwrap 能给沙箱一个**独立的 host UID**（从而让内核也参与隔离）**只有在满足条件时**：

- 容器以 root 运行 + 镜像里有 `newuidmap/newgidmap`（uidmap 包）+ `/etc/subuid`、`/etc/subgid` 配置了范围 ⇒ `bwrap --unshare-user --uid <每沙箱唯一UID>` 生效；
- 否则 bwrap 只能把"当前 uid"映射成沙箱内的 uid ⇒ **所有沙箱共用同一个宿主 uid**，隔离就完全依赖 mount namespace。

两种情况下都必须做到的事：**只挂该沙箱自己的目录**（绝不挂 `/var/sandbox/tenants` 根），并加 `--cap-drop ALL`（避免沙箱内 root 能 `umount` 掉只读绑定，从而改写模板）。

自检命令（建议做成 Pod 启动探针）：

```bash
id -u; grep -c . /etc/subuid 2>/dev/null || echo "no subuid"
command -v newuidmap || echo "no newuidmap"
bwrap --unshare-user --uid 10001 --ro-bind / / -- id -u 2>&1 | tail -1
```

---

## 5. 控制面改造

### 5.1 `api/chat_server.py`（调度服务）

```python
# 伪代码
@app.post("/chat")
async def chat(req: ChatRequest, user = Depends(auth)):
    tenant = user.tenant_id                       # ★ 身份来自鉴权，不信任 body 里的 user_id
    agent  = req.agent_name
    if not acl.can_use(tenant, agent):
        raise HTTPException(403, "agent not allowed")

    # ① 归属校验：会话必须是该租户该智能体的
    if req.session_id and not sessions.owned_by(req.session_id, tenant, agent):
        raise HTTPException(403, "session not owned")

    # ② 取/建沙箱
    sbx = await mgr.ensure(tenant, agent)          # SandboxHandle(endpoint, token, directory="/workspace")

    # ③ 转发：目录参数按"沙箱内视角"传，不要传宿主路径
    headers = {
        "authorization": "Basic " + b64(f"opencode:{sbx.token}"),
        "x-opencode-directory": "/workspace",      # ★ 沙箱内路径（见 workspace-routing.ts:37-43 的注释）
        "content-type": "application/json",
    }
    async with httpx.AsyncClient(timeout=None) as c:
        async with c.stream("POST", f"{sbx.endpoint}/session/{sid}/message", json=payload, headers=headers) as r:
            async for chunk in r.aiter_raw():
                yield chunk                            # SSE 透传
```

必须补齐的几点：

1. **目录参数必须是沙箱内路径**：中间件在**沙箱进程内**做 `FSUtil.resolve()`，传宿主路径会得到不存在的目录（上游为此外专门剥离了 `directory`，见 `server/shared/workspace-routing.ts:37-43`、`server/proxy-util.ts:17`）。
2. **会话归属**：`GET/POST /session/:id*` 一律先查控制面的 `session → (tenant, agent)` 映射；绝不把前端的 `session_id` 直接转发（原因见 1.2(5)）。
3. **事件流**：每个用户的 SSE 连接必须用他自己沙箱的 endpoint + token；`/event` 已按 directory 过滤（`handlers/event.ts:35-39`），但跨沙箱仍是不同进程，天然隔离。
4. **权限代答**：订阅沙箱事件，收到 `permission.asked` 后走策略引擎，再回调 reply；无人在线时默认拒绝。
5. **超时/取消**：客户端断开要 `POST /session/:id/abort`，避免沙箱里留 inflight 生成。
6. **不要用 v2 `/api/*` 的会话列表**：`GET /api/session` 在不带任何参数时会走到 `SessionsAllQuery` 分支，而过滤条件是"按键存在判断"（`packages/core/src/session.ts:273-277`：`if ("directory" in input)` / `if ("project" in input)`），条件为空即全库；`/experimental/session` 也明确跨项目（`groups/experimental.ts`）。多租户部署下这是直接的越权读。v1 `GET /session` 默认收敛到实例目录（`handlers/session.ts:64-75`），但仍建议只在各自的沙箱进程内调用。
   > ⚠️ **需实测**：`packages/protocol/src/groups/session.ts:100` 用的是原生 `Schema.optional`（不是 schema 包的 `optional(...)` 助手），解码缺省 query 时是否保留 `directory: undefined` 这个键会影响 v2 列表的实际行为。上线前用 `curl -u ... 'http://sandbox/api/session'` 对照本沙箱的 `GET /session?directory=...` 各跑一次确认。
7. **prompt 的响应不是 SSE**：v1 `POST /session/:id/message` 返回单块 JSON（`handlers/session.ts:295-309`），流式必须走 `/event`；异步用 `/prompt_async`（204）。别按 OpenAPI 里"streaming the AI response"的过时描述去实现（`groups/session.ts:326`）。

### 5.2 `mgr/mgr_server.py`（配置管理 + 沙箱管理）

保留原有 5 个 agent 接口，新增 sandbox 组：

| 方法 | 路径 | 语义 |
|---|---|---|
| POST | `/sandbox/ensure` | `{tenant_id, agent_name}` → `{endpoint, token, status}`；不存在则创建 |
| POST | `/sandbox/create` | 预创建（可指定配额、模板版本） |
| POST | `/sandbox/{id}/start` \| `/stop` | 显式启停（保留磁盘） |
| DELETE | `/sandbox/{id}` | 销毁：停进程 + 删除/归档租户目录 |
| GET | `/sandbox` | 列表 + 状态 + RSS/CPU/最近活跃 |
| GET | `/sandbox/{id}/logs` | 拉取最近 N 行日志 |

`cli.py` 同步加 `sandbox ensure|list|stop|rm`，便于排障。

**另外建议把"智能体配置管理"和"沙箱运行时"拆成两个服务**：前者是低频写、强一致；后者是高频、有状态、要池化。放同一个 FastAPI 进程会让重启/扩容互相牵连。

### 5.3 与 webchat 的接口契约

```jsonc
// POST /chat
{
  "conversation_id": "c_xxx",     // 前端只认这个；映射到 (tenant, agent, session_id) 由平台维护
  "agent_name": "test_agent_1",
  "message": "...",
  "stream": true
}
```

服务端返回事件中**不要**泄漏内部信息：沙箱 endpoint、端口、宿主路径、`project_id`、其他用户的 session_id。前端所有请求都不应出现 `directory`、`session_id`、`port` 这类字段。

---

## 6. 与 opencode 上游能力对齐的两条升级路径

### 6.1 用官方 Workspace 抽象做"沙箱路由"（需要较新版本）

上游已经在做"把会话放到另一个执行目标"的抽象，正好对得上你们的需求：

```ts
// packages/plugin/src/index.ts:47-66
export type WorkspaceAdapter = {
  name: string; description: string
  configure(config: WorkspaceInfo): WorkspaceInfo | Promise<WorkspaceInfo>
  create(config: WorkspaceInfo, env, from?): Promise<void>
  remove(config: WorkspaceInfo): Promise<void>
  target(config: WorkspaceInfo): WorkspaceTarget | Promise<WorkspaceTarget>   // {type:"local"|"remote"}
}
export type PluginInput = { ..., experimental_workspace: { register(type, adapter): void } }
```

```ts
// packages/opencode/src/control-plane/adapters/index.ts:37-41（插件注册进 per-project 适配表）
// packages/opencode/src/control-plane/types.ts:30-39（Target = local{dir} | remote{url,headers}）
// packages/opencode/src/control-plane/workspace.ts:492-557（create：configure→DB→create(env)→startSync）
// .../middleware/workspace-routing.ts:113-146（remote 目标：HTTP/WS 代理 + Fence 同步栅栏）
```

落地方式：写一个插件，注册 `type: "sandbox"` 适配器：

- `configure()`：分配 `tenant/agent` 目录、端口、口令；
- `create()`：`bwrap` 起沙箱（或调 K8s API 创建 Sandbox）；
- `target()`：返回 `{type:"remote", url:"http://127.0.0.1:<port>", headers:{authorization: basic}}`；
- `remove()`：停进程 + 归档。

这样上游的 session→workspace 绑定、`/experimental/workspace/warp`（把会话迁移到新沙箱）、SSE 转发、事件同步都免费复用。

**前置校验**（版本相关，务必先验）：

```bash
opencode --version
grep -r "experimental_workspace" node_modules/@opencode-ai/plugin/dist 2>/dev/null
curl -s -u opencode:$PWD http://127.0.0.1:17001/experimental/workspace/adapter | jq
```

若当前发布的二进制没有这些能力，则按第 4、5 节自建控制面——**两条路互不冲突**：自建沙箱管理器仍然是必需的（创建/回收沙箱），差别只在"路由是谁做的"。

### 6.2 K8s 原生：agent-sandbox（方案 C）

- **`Sandbox` CRD**（`agents.x-k8s.io`）：单容器、有状态、稳定身份；`SandboxTemplate` + `SandboxClaim` + **`SandboxWarmPool`** 提供预热池，解决"按需起 Pod 秒级延迟"问题；`Sandbox Router` 提供到沙箱 Pod 的 HTTP 反代（gVisor/Kata 下 port-forward 不可用时尤其有用）。参考：<https://agent-sandbox.sigs.k8s.io/docs/getting_started/overview/>。
- **强隔离**：`RuntimeClass` 指向 gVisor 或 Kata Containers（该项目本身把低层隔离委托给这些运行时）。
- **网络**：每租户 Pod → `NetworkPolicy`/Cilium FQDN 白名单，只放行 LLM 网关与必要的包源。
- **资源**：Pod `requests/limits` + QoS + `LimitRange`/`ResourceQuota`；`hostUsers: false`（K8s ≥1.36 GA）进一步做用户命名空间隔离。
- **存储**：每租户 PVC（`volumeClaimTemplates`），或从 WarmPool 里"认领 + 挂载"。
- 控制面改动：`ensure(tenant, agent)` 从"fork 进程"变成"claim 一个 Sandbox 并等 Ready"，`endpoint` 从 `127.0.0.1:port` 变成 `http://sandbox-<id>:port`（或经 Sandbox Router）。

### 6.3 如果要改 opencode 源码：先认清 V1 / V2 双实现

这个仓库正处于新老架构并行期，**改造前必须确认你要动的面**：

| | V1 legacy | V2 core |
|---|---|---|
| 代码 | `packages/opencode/src/{tool,session,permission,agent}` | `packages/core/src/{tool,permission,filesystem}` + `packages/protocol` + `packages/server` |
| HTTP | `/session`、`/agent`、`/config`…（`opencode serve` 默认，也是你现在用的面） | `/api/*` |
| 文件系统保护 | `FSUtil` + 事后 `assertExternalDirectory`（**无 jail**） | `FileSystem.resolve` 有 realpath 双重校验（`packages/core/src/filesystem.ts:65-73` 的 "Path escapes the location"）与 `LocationMutation` 的 `relative_escape`/`location_escape`（`packages/core/src/location-mutation.ts:120-150`） |
| 权限 | `permission/pattern/action` 规则表 + `permission.asked` 事件 | `action/resource/effect` 结构化规则 + 可持久化 saved 规则（`packages/core/src/permission.ts:190-218`） |
| 工具注册 | 目录（Instance）级 | Location 级 + 进程级 `ApplicationTools` |
| bash 隔离 | 无（`tool/shell.ts:293-310`） | 同样无（`core/src/tool/bash.ts:109` 自述 "the host user's filesystem, process, and network authority"） |
| 缺口 | — | V2 尚缺 `task`/LSP 等工具（`core/src/tool/builtins.ts:26-29` TODO） |

结论：**生产稳定优先 → 继续走 V1 的 `serve` + 进程外沙箱**（本文方案 B/C）；**要把隔离做进 opencode 内部 → V2 是更好的基座**（已有路径 jail 与结构化权限），但要接受工具集不完整、Session V2 仍在演进的风险。

上游对"bash 不是沙箱"是明确的（`specs/v2/session.md:204`、`core/src/tool/bash.ts:109`），所以任何"在 opencode 内部做沙箱"的改造都属于**新增能力**，不是修 bug——需要自己维护 fork 或走插件/适配器扩展点。

---

## 7. 落地路线图

| 阶段 | 内容 | 产出 | 预估 |
|---|---|---|---|
| **P0 救火** | 每 (user, agent) 独立目录 + 独立 `XDG`/`OPENCODE_DB` + 每实例口令 + `OPENCODE_DISABLE_PROJECT_CONFIG=1`；修 `cwd` bug、去掉 `NODE_TLS_REJECT_UNAUTHORIZED=0` | 会话不再串、文件不再互相覆盖（**仍非安全边界**） | 1–3 人日 |
| **P1 控制面** | `SandboxManager`（ensure/start/stop/destroy/list、端口池、健康检查、空闲回收、并发锁）；`chat_server` 路由与归属校验；`cli.py` 子命令 | 可按用户灰度、可运维 | 1–2 周 |
| **P2 沙箱硬化** | `bwrap` 包裹 + 只读模板 + 权限规则集 + 策略引擎代答 + 日志/配额 + 攻击用例回归 | 单 Pod 内多租户可用 | 2–4 周 |
| **P3 集群化** | agent-sandbox + WarmPool + RuntimeClass(gVisor/Kata) + NetworkPolicy + 每租户 PVC/配额；灰度替换 P2 后端 | 强隔离生产形态 | 4–8 周（含集群能力准备） |
| 并行 | 观测：每沙箱 RSS/CPU/请求数/拒绝数；审计：命令、文件变更、出网 | 安全与容量运营 | 持续 |

**每阶段验收门槛**：第 8 节攻击用例全部为"应该失败"，且正常业务用例（多用户并发、长会话、技能调用、代码读写）全部通过。

---

## 8. 安全验收清单（务必逐条实测）

| # | 攻击/异常用例 | 期望结果 | 主要依赖 |
|---|---|---|---|
| 1 | 用户 A 的 agent 执行 `cat /var/sandbox/tenants/B/**` | 路径不存在（未挂载） | mount ns |
| 2 | `ls /proc/*/cmdline` 看到其他租户进程 | 看不到 | `--unshare-pid` |
| 3 | `kill -9 <B 的 pid>` | 无效 | pid ns + 独立 UID |
| 4 | 读取 `/etc/shadow`、`/root/.ssh`、SA token | 不可读/不存在 | 最小 ro-bind + `automountServiceAccountToken:false` |
| 5 | `curl http://127.0.0.1:<B 的端口>/session` | 401（口令不同） | 每沙箱口令 |
| 6 | 用 B 的 `session_id` 请求 `/chat` | 403（控制面归属校验） | 控制面 |
| 7 | 在工作区放 `opencode.json` / `.opencode/plugin/x.ts` 注入 | 不生效、不执行 | `OPENCODE_DISABLE_PROJECT_CONFIG=1` |
| 8 | 让 agent 读宿主机 `/opt/agents/<other>` | 只读模板可见但不含用户数据（模板本身非机密） | 设计约定 |
| 9 | `:(){ :|:& };:` fork bomb | 进程数被 `ulimit -u`/prlimit 限制，Pod 不受影响 | rlimit（硬配额需 P3） |
| 10 | 写满磁盘 | 配额/轮转生效，Pod 不 OOM | 磁盘配额/日志轮转 |
| 11 | 连接内网其他服务 / 扫描网段 | 单 Pod 内受限（软），P3 后应完全阻断 | NetworkPolicy（P3） |
| 12 | 从沙箱读取 LLM API key | 读不到（只有短期网关 token） | 凭据网关 |
| 13 | 沙箱进程崩溃/OOM | 自动重启且会话可恢复（DB 完好） | 控制面 + SQLite WAL |
| 14 | 100 并发用户同时首问 | 沙箱按需创建、有上限保护、不雪崩 | 池化 + 排队 |
| 15 | 通过 PTY / `!command` / 插件 `Bun.$` / MCP 执行命令 | 同样被限制在沙箱内（或这些入口已被网关关闭） | 进程级沙箱（**只包 bash 无效**） |
| 16 | 调 `GET /api/session`（不带参数）试图列出全库会话 | 网关层拦截 / 不暴露该面 | 控制面（`core/src/session.ts:291-302`） |
| 17 | 调 `POST /global/dispose` | 403 | 控制面 |

---

## 9. 明确"不要做"的事

1. **不要**把"单进程 + `x-opencode-directory`"当作安全边界（§1.2(2)(5)、§3.2 A 方案）。
2. **不要**依赖 `permission`/`external_directory` 当沙箱：它是"询问式"，被 allow 或未被 AST 识别的命令即可绕过（`tool/shell.ts:390-411`、`specs/v2/session.md:204`）；它也不是内核级 jail。
3. **不要**让用户可写目录参与配置/插件发现（`config.ts:420-479`）。
4. **不要**只包 bash 工具就以为隔离完成：PTY、`!command`、插件 `Bun.$`、MCP、LSP 都不走权限系统（§1.2(10)）。
5. **不要**用插件钩子 `permission.ask` 做审批网关（死代码，§4.6）。
6. **不要**在同一进程内把不同用户指向同一目录，也不要指望 opencode 的 API 自带租户过滤（§1.2(5)、§5.1(6)）。
7. **不要在**沙箱内挂载 Pod 的 SA token、宿主机 docker socket、全量 PVC（`packages/skills` 只读子路径即可）。
8. **不要**所有沙箱复用口令、不要把 serve 暴露到 `0.0.0.0`/开 mDNS。
9. **不要**在用户可见的返回里泄漏 endpoint、端口、宿主路径、其他 session_id。
10. **不要**为了省事开启 TLS 校验豁免（`NODE_TLS_REJECT_UNAUTHORIZED=0`）。

---

## 10. 附录：源码索引

| 主题 | 位置 |
|---|---|
| serve 无实例启动、按请求装载 | `packages/opencode/src/cli/cmd/serve.ts:10-12` |
| 目录来源（头/查询/ cwd） | `packages/opencode/src/server/routes/instance/httpapi/middleware/workspace-routing.ts:86-88` |
| 实例装载与上下文注入 | `.../middleware/instance-context.ts:27-34` |
| 实例缓存（realpath 键，无 TTL） | `packages/opencode/src/project/instance-store.ts:43,108-124` |
| 路径边界判定 | `packages/opencode/src/project/instance-context.ts:18-24` |
| project id 解析（git remote/root commit/global） | `packages/core/src/project.ts:73-122` |
| 会话表结构（project_id/directory/workspace_id） | `packages/core/src/session/sql.ts:22-66` |
| 会话列表按目录过滤 | `packages/opencode/src/session/session.ts:546-553` |
| 事件流按目录过滤 | `.../httpapi/handlers/event.ts:35-39` |
| 权限询问/回复 | `packages/opencode/src/permission/index.ts:67-120` |
| `permission.ask` 插件钩子是死代码 | 仅 `packages/plugin/src/index.ts:261` 有声明，全仓无 trigger |
| 默认权限规则（`"*": "allow"`） | `packages/opencode/src/agent/agent.ts:108-136` |
| 绕过权限的执行入口（PTY / `!command` / `Bun.$` / MCP） | `packages/core/src/pty.ts:165-183`、`packages/opencode/src/session/prompt.ts:552-578`、`packages/opencode/src/plugin/index.ts:167` |
| V2 的路径 jail（可照搬的实现） | `packages/core/src/filesystem.ts:65-73`、`packages/core/src/location-mutation.ts:120-150` |
| 会话列表"不带参数返回全库" | `packages/core/src/session.ts:291-302`、`packages/protocol/src/groups/session.ts:55-59,98-104` |
| `XDG_*` 导入期求值（必须进程级隔离） | `packages/core/src/global.ts:11-15`、`packages/opencode/test/preload.ts:1-2` |
| bash 工具 spawn | `packages/opencode/src/tool/shell.ts:293-310,416-426,481-484` |
| `config.shell` wrapper 可行性 | `packages/opencode/src/tool/shell.ts:600`、`packages/core/src/shell.ts:13-23,114-121,183-196` |
| Config 是**按目录**隔离（InstanceState） | `packages/opencode/src/config/config.ts:614-616` |
| 按目录隔离的服务清单（23 处 `InstanceState.make`） | `agent/agent.ts:98`、`skill/index.ts:259,273`、`plugin/index.ts:134`、`permission/index.ts:46`、`tool/registry.ts:121`、`lsp/lsp.ts:145`、`mcp/index.ts:492`、`snapshot/index.ts:66`、`provider/provider.ts:1451` … |
| 每智能体主提示词（config + `{file:}`） | `packages/core/src/v1/config/agent.ts:20`、`packages/opencode/src/agent/agent.ts:283`、`session/llm/request.ts:60`、`config/variable.ts:33-90` |
| 配置向上发现**包含起始目录** | `packages/core/src/fs-util.ts:168-182` |
| `GET /session` 不传 directory 时只按 project 过滤 | `.../httpapi/handlers/session.ts:64-75` |
| 每目录资源回收 `POST /instance/dispose` | `.../httpapi/groups/instance.ts:44,62-71`、`handlers/instance.ts:24-27` |
| snapshot 影子库按 (project, worktree) 分 | `packages/opencode/src/snapshot/index.ts:71` |
| 插件模块缓存是进程级 ES module cache | `packages/opencode/src/plugin/loader.ts:139` |
| session→directory 绑定（v1 路由） | `.../middleware/workspace-routing.ts:181-184,212-235`、`server/shared/workspace-routing.ts:5-9,20-29` |
| v2 PermissionSaved 按 project 落库 | `packages/core/src/permission/saved.ts:46`、`packages/core/src/permission.ts:250-256` |
| 全局 `AGENTS.md` 进入所有目录系统提示 | `packages/opencode/src/session/instruction.ts:60-63,115-120` |
| MCP OAuth 回调是进程级单例 | `packages/opencode/src/mcp/oauth-callback.ts:9-22` |
| v1 runner 按目录（同 session 可被两目录并发驱动） | `packages/opencode/src/session/run-state.ts:35-49` |
| location 服务 60 分钟 TTL（配置变更需 dispose） | `packages/core/src/location-services.ts:109`、`packages/opencode/src/effect/instance-state.ts:38` |
| `shell.env` 插件钩子 | `packages/opencode/src/tool/shell.ts:416-426` |
| 插件系统与 workspace adapter 注册 | `packages/plugin/src/index.ts:47-66`、`packages/opencode/src/plugin/index.ts:159-161` |
| Workspace 创建/代理/warp | `packages/opencode/src/control-plane/workspace.ts:492-557`、`.../middleware/workspace-routing.ts:113-158` |
| DB 路径（OPENCODE_DB / XDG） | `packages/core/src/database/database.ts:43-55`、`packages/core/src/global.ts:3-14` |
| 环境变量总表 | `packages/core/src/flag/flag.ts` |
| 配置合并顺序 | `packages/opencode/src/config/config.ts:412-490`、`config/paths.ts:10-41` |
| "bash 不是沙箱"的官方说明 | `specs/v2/session.md:204` |
| 远端代理剥离 directory 的教训 | `packages/opencode/src/server/shared/workspace-routing.ts:37-43`、`server/proxy-util.ts:17` |
| 工具输出/日志截断（磁盘保护参考） | `packages/opencode/src/tool/truncate.ts`、`tool/truncation-dir.ts` |

外部参考：

- [Kubernetes v1.36：用户命名空间正式可用](https://kubernetes.io/zh-cn/blog/2026/04/23/userns-ga/)
- [Agent Sandbox 概览（Sandbox / SandboxTemplate / SandboxClaim / SandboxWarmPool / Router）](https://agent-sandbox.sigs.k8s.io/docs/getting_started/overview/)
- [在 Kubernetes 上使用 Agent Sandbox 运行智能体](https://kubernetes.io/zh-cn/blog/2026/03/20/running-agents-on-kubernetes-with-agent-sandbox/)
- [opencode issue：如何给 agent 加沙箱](https://github.com/anomalyco/opencode/issues/2242)
