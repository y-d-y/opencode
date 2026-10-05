# opencode.db 表结构与字段说明（opencode 1.18.34）

> 权威来源：
> - 新库全量建表 DDL：`packages/core/src/database/schema.gen.ts`（274 行，包含全部 `CREATE TABLE` 与 `CREATE INDEX`）
> - 各表 Drizzle 定义：`packages/core/src/**/sql.ts`
> - 迁移与建库逻辑：`packages/core/src/database/migration.ts` + `migration.gen.ts`（38 个迁移）
> - 列的自定义类型（绝对路径校验等）：`packages/core/src/database/path.ts`
>
> 说明：本文按"**一个新装实例的库**"为准；老库由 38 个迁移逐步演进而来，可能残留历史表（如 `__drizzle_migrations`）。

---

## 1. 库文件位置与命名

```ts
// packages/core/src/database/database.ts:43-55
export function path() {
  if (Flag.OPENCODE_DB) {
    if (Flag.OPENCODE_DB === ":memory:" || isAbsolute(Flag.OPENCODE_DB)) return Flag.OPENCODE_DB
    return join(Global.Path.data, Flag.OPENCODE_DB)          // 相对路径 → 拼到 data 目录
  }
  if (["latest", "beta", "prod"].includes(InstallationChannel) ||
      process.env.OPENCODE_DISABLE_CHANNEL_DB === "1" || process.env.OPENCODE_DISABLE_CHANNEL_DB === "true")
    return join(Global.Path.data, "opencode.db")
  return join(Global.Path.data, `opencode-${InstallationChannel...}.db`)   // 非正式 channel 带后缀
}
```

| 项 | 值 |
|---|---|
| 默认路径 | `$XDG_DATA_HOME/opencode/opencode.db`（即 `~/.local/share/opencode/opencode.db`） |
| 非 latest/beta/prod channel | `opencode-<channel>.db`（例如本地构建 `opencode-local.db`） |
| 覆盖方式 | `OPENCODE_DB=<绝对路径>` / `OPENCODE_DB=:memory:` / `OPENCODE_DB=<相对 data 的文件名>` |
| 驱动 | `bun:sqlite`（Bun）或 `node:sqlite`（Node），条件导入 `#sqlite` |

**连接参数**（每次打开都设置，`database.ts:27-33`）：

```sql
PRAGMA journal_mode = WAL;        -- 预写日志
PRAGMA synchronous = NORMAL;
PRAGMA busy_timeout = 5000;
PRAGMA cache_size = -64000;       -- 64MB
PRAGMA foreign_keys = ON;         -- ★ 外键级联生效的前提
PRAGMA wal_checkpoint(PASSIVE);
```

> ⚠️ 运维含义：`foreign_keys=ON` 是**每个连接**的开关，所以 `ON DELETE CASCADE` 只有通过 opencode 自身连接才生效。用外部工具（sqlite3 CLI）删 `session` 行时，默认**不会**级联删除 `message`/`part`——远程备份/清理脚本务必 `PRAGMA foreign_keys=ON` 或手工级联。

---

## 2. 建库与迁移机制

```ts
// packages/core/src/database/migration.ts:18-41
const tables = await db.all(`SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%'`)
if (tables.some(t => t.name === "session")) return applyOnly(db, migrations)   // 老库：只跑未完成的迁移
if (tables.length > 0) return Effect.die("Database is not empty and has no session table")  // 非空但无 session → 直接失败
// 新库：一次性执行 schema.gen 全量 DDL + 建 migration 表 + 写入全部迁移 id
```

- **判定表存在与否的锚点是 `session` 表**：空目录 → 建全量 schema；有 `session` → 走增量迁移；有其他表但无 `session` → 进程 die。
- `migration` 表是**迁移台账**（不是业务表），由 `migration.ts` 而非 `schema.gen.ts` 创建。
- 老库若只有 Drizzle 的 `__drizzle_migrations`，会做一次 journal 播种（`migration.ts:51-93`），避免重放历史 SQL。
- 迁移清单见 `packages/core/src/database/migration.gen.ts:5-42`，共 **38 个**，跨度 `20260127222353_familiar_lady_ursula` → `20260622202450_simplify_session_input`。

---

## 3. 表总览（20 张：19 业务 + 1 台账）

| # | 表 | 归属 | 用途 | 行量级 |
|---|---|---|---|---|
| 1 | `project` | 共享 | 项目（工作区）身份，`id` 来自 git remote/root commit | 个位~百 |
| 2 | `project_directory` | 共享 | 一个项目下的多个检出目录 | 项目数×目录数 |
| 3 | `workspace` | V2 | 执行目标（git worktree / 远端沙箱） | 小 |
| 4 | **`session`** | **共享** | **会话**（父/子会话都在这） | 用户数×会话数 |
| 5 | **`message`** | V1 | 消息（正文在 JSON） | 会话数×轮数 |
| 6 | **`part`** | V1 | 消息的分片（文本/推理/工具调用…） | 消息数×part 数 |
| 7 | `todo` | V1 | 会话待办 | 小 |
| 8 | `session_share` | V1 | 会话分享链接 | 小 |
| 9 | `session_message` | **V2 专属** | 事件投影出的会话消息（`seq` 排序） | 与 `message` 同量级 |
| 10 | `session_input` | **V2 专属** | 持久化 inbox（admitted/promoted） | 轮数 |
| 11 | `session_context_epoch` | **V2 专属** | System Context 基线 + 比较快照 | 每会话 1 行 |
| 12 | `event` | V2 | 事件溯源日志 | **最大表**，随每次会话动作增长 |
| 13 | `event_sequence` | V2 | 每个聚合（=session）的序号水位 | 每会话 1 行 |
| 14 | `permission` | V2 | 「始终允许」等已保存权限（按 project） | 小 |
| 15 | `account` | 账号 | 控制台账号（OAuth token） | 小 |
| 16 | `account_state` | 账号 | 当前激活账号/组织（单行） | 1 |
| 17 | `control_account` | **LEGACY** | 旧版账号表 | 小 |
| 18 | `credential` | 集成 | 集成凭据（按 integration/connector） | 小 |
| 19 | `data_migration` | 台账 | JSON→SQLite 数据迁移进度 | 小 |
| — | `migration` | 台账 | SQL 迁移台账（`migration.ts` 创建） | 38 行 |

> 另有**不属于 DB** 但和上下文成本直接相关的落盘数据：工具输出截断写在 `<XDG_DATA_HOME>/opencode/tool-output/`（`packages/opencode/src/tool/truncation-dir.ts:4`），会话快照影子库在 `<XDG_DATA_HOME>/opencode/snapshot/<project.id>/<hash(worktree)>`（`packages/opencode/src/snapshot/index.ts:71`）。

---

## 4. 核心表字段详解

### 4.1 `session` —— 会话（上下文与计费的锚点）

```sql
CREATE TABLE `session` (
  `id` text PRIMARY KEY,
  `project_id` text NOT NULL,
  `workspace_id` text,
  `parent_id` text,
  `slug` text NOT NULL,
  `directory` text NOT NULL,
  `path` text,
  `title` text NOT NULL,
  `version` text NOT NULL,
  `share_url` text,
  `summary_additions` integer, `summary_deletions` integer, `summary_files` integer, `summary_diffs` text,
  `metadata` text,
  `cost` real DEFAULT 0 NOT NULL,
  `tokens_input` integer DEFAULT 0 NOT NULL,
  `tokens_output` integer DEFAULT 0 NOT NULL,
  `tokens_reasoning` integer DEFAULT 0 NOT NULL,
  `tokens_cache_read` integer DEFAULT 0 NOT NULL,
  `tokens_cache_write` integer DEFAULT 0 NOT NULL,
  `revert` text, `permission` text, `agent` text, `model` text,
  `time_created` integer NOT NULL, `time_updated` integer NOT NULL,
  `time_compacting` integer, `time_archived` integer,
  CONSTRAINT fk_session_project_id_project_id_fk FOREIGN KEY (`project_id`) REFERENCES `project`(`id`) ON DELETE CASCADE
);
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | text PK | `ses_` 前缀。生成方式有两种：`SessionID.descending()`（V1 创建，`session.ts:513`）与 V2 的 `ascending`；都是 `ses_` + 12 位十六进制时间 + 14 位随机 base62（`packages/schema/src/identifier.ts:14-29`） |
| `project_id` | text NOT NULL → `project.id` | 级联删除。**注意 `project.id` 由 git 身份决定，会跨用户碰撞**（`core/src/project.ts:110-122`） |
| `workspace_id` | text NULL | 绑定的执行目标（V2 workspace），有索引 |
| `parent_id` | text NULL | **非空 = 子会话**（task 派生的 subagent 会话）。有索引；用于 `GET /session?roots=true` 过滤 |
| `slug` | text NOT NULL | 人类可读短名（`Slug.create()`） |
| `directory` | text NOT NULL | 会话的工作目录，**绝对路径**（自定义列校验绝对性并做平台归一化，`database/path.ts:45-59`）。**权限与上下文边界都基于它** |
| `path` | text NULL | 工作区内的相对路径（用于"同一 project 下按子路径分组"） |
| `title` | text NOT NULL | 标题；默认 `"New session - <ISO>"`，子会话为 `"Child session - <ISO>"`，由 `title` agent 异步覆盖 |
| `version` | text NOT NULL | **创建该会话的 opencode 版本**（排查兼容问题用） |
| `share_url` | text NULL | 分享链接（另有 `session_share` 表存 secret） |
| `summary_additions/deletions/files` | integer | 会话内文件变更统计（由 `SessionSummary.summarize` 写） |
| `summary_diffs` | text JSON | `Snapshot.LegacyFileDiff[]` |
| `metadata` | text JSON | `Record<string, unknown>` —— **平台可用的自由扩展位**（当前 V1 不解释内容） |
| `cost` | real | 累计费用 |
| `tokens_input/output/reasoning/cache_read/cache_write` | integer | 累计 token（`isOverflow` 判定的数据来源之一） |
| `revert` | text JSON | `Revert.State`（`{ messageID, partID? }`）；非空时下一轮 prompt 会先做物理删除 |
| `permission` | text JSON | 会话级权限规则集（`{permission, pattern, action}[]`），由 `POST /session` / `PATCH` 或 prompt 的 `tools` 参数写入 |
| `agent` | text NULL | 会话绑定的 agent（子会话创建时写入，用于导航/展示） |
| `model` | text JSON | `{ id, providerID, variant? }`（最近使用的模型） |
| `time_created/updated` | integer | epoch ms |
| `time_compacting` | integer NULL | **V1 代码里没有任何写入点**（仅 schema/DB/TUI 读取），当前是死字段 |
| `time_archived` | integer NULL | 归档时间；列表查询默认排除 |

**索引**：`session_project_idx(project_id)`、`session_workspace_idx(workspace_id)`、`session_parent_idx(parent_id)`。

### 4.2 `message` / `part` —— V1 的消息与分片

```sql
CREATE TABLE `message` (
  `id` text PRIMARY KEY, `session_id` text NOT NULL,
  `time_created` integer NOT NULL, `time_updated` integer NOT NULL,
  `data` text NOT NULL,
  FOREIGN KEY (`session_id`) REFERENCES `session`(`id`) ON DELETE CASCADE
);
CREATE TABLE `part` (
  `id` text PRIMARY KEY, `message_id` text NOT NULL, `session_id` text NOT NULL,
  `time_created` integer NOT NULL, `time_updated` integer NOT NULL,
  `data` text NOT NULL,
  FOREIGN KEY (`message_id`) REFERENCES `message`(`id`) ON DELETE CASCADE
);
```

| 列 | 说明 |
|---|---|
| `message.id` | `msg_` 前缀 |
| `message.session_id` | 级联删除；**注意 `sessionID` 也冗余存在 `data` 里**（表列是权威） |
| `message.data` | JSON = `SessionV1.User` 或 `SessionV1.Assistant` 去掉 `id`/`sessionID`（`Omit<SessionV1.Info,"id"\|"sessionID">`） |
| `part.id` | `prt_` 前缀 |
| `part.message_id` | 级联删除（删消息即删所有 part） |
| `part.session_id` | 冗余列，便于按会话直接查（有独立索引） |
| `part.data` | JSON = `SessionV1.Part` 去掉 `id`/`sessionID`/`messageID` |

**索引**：`message_session_time_created_id_idx(session_id, time_created, id)`（历史分页游标）、`part_message_id_id_idx(message_id, id)`、`part_session_idx(session_id)`。

### 4.3 `todo` / `session_share`

```sql
CREATE TABLE `todo` (
  `session_id` text NOT NULL, `content` text NOT NULL, `status` text NOT NULL,
  `priority` text NOT NULL, `position` integer NOT NULL,
  `time_created` integer NOT NULL, `time_updated` integer NOT NULL,
  PRIMARY KEY(`session_id`, `position`),
  FOREIGN KEY (`session_id`) REFERENCES `session`(`id`) ON DELETE CASCADE
);
CREATE TABLE `session_share` (
  `session_id` text PRIMARY KEY, `id` text NOT NULL, `secret` text NOT NULL,
  `url` text NOT NULL, `time_created` integer NOT NULL, `time_updated` integer NOT NULL,
  FOREIGN KEY (`session_id`) REFERENCES `session`(`id`) ON DELETE CASCADE
);
```

- `todo`：**复合主键 `(session_id, position)`**，`position` 是排序位；`status` 常见 `pending/in_progress/completed`，`priority` 常见 `high/medium/low`。
- `session_share`：会话级 1:1；`secret` 是分享凭据，`url` 是公开地址。**多租户下这张表是数据外泄面**，销户/停用时别忘清理。

### 4.4 `project` / `project_directory` / `workspace`

```sql
CREATE TABLE `project` (
  `id` text PRIMARY KEY, `worktree` text NOT NULL, `vcs` text, `name` text,
  `icon_url` text, `icon_url_override` text, `icon_color` text,
  `time_created` integer NOT NULL, `time_updated` integer NOT NULL, `time_initialized` integer,
  `sandboxes` text NOT NULL, `commands` text
);
CREATE TABLE `project_directory` (
  `project_id` text NOT NULL, `directory` text NOT NULL, `type` text, `strategy` text,
  `time_created` integer NOT NULL,
  PRIMARY KEY(`project_id`, `directory`),
  FOREIGN KEY (`project_id`) REFERENCES `project`(`id`) ON DELETE CASCADE
);
```

| 表.字段 | 说明 |
|---|---|
| `project.id` | **git 身份**：`hash("git-remote:"+归一化remote)` → `.git/opencode` 缓存 → root commit SHA → 兜底 `"global"`（`core/src/project.ts:73-122`）。**非 git 项目全部塌缩为 `"global"`** |
| `project.worktree` | 绝对路径；非 git 项目被置为 `"/"` |
| `project.vcs` | JSON `{ type: "git", store: <git common dir 绝对路径> }` |
| `project.sandboxes` | JSON 数组（绝对路径），同一 project 的其他检出目录 |
| `project.commands` | JSON `{ start?: string }`（项目自定义命令） |
| `project.time_initialized` | `/init` 执行过的时间 |
| `project_directory.type` | `"main"` \| `"root"` \| `"git_worktree"` |
| `project_directory.strategy` | 目录复制策略 id（如 `"git_worktree"`） |
| `workspace` | 见 `core/src/control-plane/workspace.sql.ts`：`id(wrk_)/type/name/branch/directory/extra(JSON)/project_id FK/time_used`，`extra` 供适配器存自定义状态 |

### 4.5 V2 事件与投影表

```sql
CREATE TABLE `event_sequence` (`aggregate_id` text PRIMARY KEY, `seq` integer NOT NULL, `owner_id` text);
CREATE TABLE `event` (
  `id` text PRIMARY KEY, `aggregate_id` text NOT NULL, `seq` integer NOT NULL,
  `type` text NOT NULL, `data` text NOT NULL,
  FOREIGN KEY (`aggregate_id`) REFERENCES `event_sequence`(`aggregate_id`) ON DELETE CASCADE
);
CREATE TABLE `session_message` (
  `id` text PRIMARY KEY, `session_id` text NOT NULL, `type` text NOT NULL, `seq` integer NOT NULL,
  `time_created` integer NOT NULL, `time_updated` integer NOT NULL, `data` text NOT NULL,
  FOREIGN KEY (`session_id`) REFERENCES `session`(`id`) ON DELETE CASCADE
);
CREATE TABLE `session_input` (
  `id` text PRIMARY KEY, `session_id` text NOT NULL, `prompt` text NOT NULL, `delivery` text NOT NULL,
  `admitted_seq` integer NOT NULL, `promoted_seq` integer, `time_created` integer NOT NULL,
  FOREIGN KEY (`session_id`) REFERENCES `session`(`id`) ON DELETE CASCADE
);
CREATE TABLE `session_context_epoch` (
  `session_id` text PRIMARY KEY, `baseline` text NOT NULL, `snapshot` text NOT NULL,
  `baseline_seq` integer NOT NULL,
  FOREIGN KEY (`session_id`) REFERENCES `session`(`id`) ON DELETE CASCADE
);
```

| 表 | 关键字段语义 |
|---|---|
| `event_sequence` | `aggregate_id` = sessionID（也用于 workspace 同步）；`seq` 是单调递增水位；`owner_id` 是同步/所有权标记 |
| `event` | 事件溯源日志：`type` 如 `session.next.prompt.admitted`、`session.next.prompted`、`session.next.context.updated`、`compaction.started/ended`、`agent.switched`…；`data` 是事件负载 JSON；**`seq` 在 `aggregate_id` 内唯一** |
| `session_message` | 由 `event` 投影而来（`projector.ts:192-208` 用 `event.durable.seq` 作 `seq`）。`type` 有 8 种：`agent-switched`、`model-switched`、`user`、`synthetic`、`system`、`shell`、`assistant`、`compaction` |
| `session_input` | durable inbox：`admitted_seq` 有值=已受理；`promoted_seq` 有值=**已进入可见历史**（其值就是那条 user 消息在 `session_message.seq` 的位置）；`delivery` = `steer` \| `queue`；NULL 的 `promoted_seq` 表示**对模型不可见** |
| `session_context_epoch` | `baseline`=冻结的 system 基线文本；`snapshot`=模型不可见的比较快照（`Record<Key,{value,removed?}>`）；`baseline_seq`=水位，"seq ≤ 它 的 system 消息已被 baseline 涵盖" |

**索引**：`event_aggregate_seq_idx`（唯一）、`event_aggregate_type_seq_idx`、`session_message_session_seq_idx`（唯一）、`session_message_session_type_seq_idx`、`session_message_session_time_created_id_idx`、`session_message_time_created_idx`、`session_input_*` 三个（1 普通 + 2 唯一）。

### 4.6 `permission`

```sql
CREATE TABLE `permission` (
  `id` text PRIMARY KEY, `project_id` text NOT NULL, `action` text NOT NULL, `resource` text NOT NULL,
  `time_created` integer NOT NULL, `time_updated` integer NOT NULL,
  UNIQUE (`project_id`,`action`,`resource`),
  FOREIGN KEY (`project_id`) REFERENCES `project`(`id`) ON DELETE CASCADE
);
```

- 这是 V2 的"已保存权限"（`PermissionSaved`）：`action` = `allow`/`deny` 等，`resource` = 工具/资源模式。
- ⚠️ **按 `project_id` 而非按目录**：`project.id` 相同的多个目录会共享"始终允许"记忆——非 git 项目全部塌缩成 `"global"` 时尤其要注意。

### 4.7 账号与凭据

| 表 | 字段 | 说明 |
|---|---|---|
| `account` | `id`, `email`, `url`, `access_token`, `refresh_token`, `token_expiry`, `time_created/updated` | 控制台账号 OAuth token；**明文存储**，库文件即机密 |
| `account_state` | `id`(integer PK，实际单行), `active_account_id` FK→`account` ON DELETE **SET NULL**, `active_org_id` | 当前激活账号/组织 |
| `control_account` | `(email,url)` 复合 PK, tokens, `active`(bool), timestamps | **LEGACY**，仅为老库保留 |
| `credential` | `id`, `integration_id`, `label`, `value`(JSON), `connector_id`, `method_id`, `active`(bool), timestamps | 集成凭据；`value` 是 JSON |

> 还有两个**不在 SQLite 里**的凭据文件：`<XDG_DATA_HOME>/opencode/auth.json`（Provider API key，0600）与 `mcp-auth.json`（MCP OAuth）。多租户隔离时要和 DB 一起按租户切。

### 4.8 台账表

| 表 | 字段 | 说明 |
|---|---|---|
| `migration` | `id` text PK, `time_completed` integer | SQL 迁移台账（由 `migration.ts` 创建，非 `schema.gen.ts`） |
| `data_migration` | `name` text PK, `time_completed` integer | JSON→SQLite **数据**迁移进度（与上者不同） |
| `__drizzle_migrations` | 历史产物 | 老库可能残留，用于一次性播种 `migration` |

---

## 5. JSON 列结构详解

### 5.1 `session.*`

| 列 | 结构 |
|---|---|
| `metadata` | `Record<string, unknown>`（任意 JSON） |
| `permission` | `{ permission: string; pattern: string; action: "ask"\|"allow"\|"deny" }[]` |
| `model` | `{ id: string; providerID: string; variant?: string }` |
| `revert` | `{ messageID: string; partID?: string }` |
| `summary_diffs` | `{ file: string; additions: number; deletions: number; ... }[]`（LegacyFileDiff） |

### 5.2 `message.data`（`role` 作为判别字段）

```jsonc
// role = "user"   （SessionV1.User 去掉 id/sessionID）
{
  "role": "user",
  "time": { "created": 1730000000000 },
  "format": { "type": "text" } | { "type": "json_schema", "schema": {...} },   // 可选
  "summary": { "title"?:"", "body"?:"", "diffs": [...] },                       // 可选
  "agent": "build",                                    // ★ 本轮用哪个 agent
  "model": { "providerID": "anthropic", "modelID": "claude-...", "variant"?:"..." },
  "system": "客户端传入的额外 system",                   // 可选
  "tools": { "write": false }                          // 可选，按消息禁用工具
}

// role = "assistant"
{
  "role": "assistant",
  "time": { "created": 1730000000000, "completed"?: 1730000001000 },
  "error"?: { "name": "APIError", "data": {...} },      // AuthError/OutputLength/Aborted/StructuredOutput/ContextOverflow/ContentFilter/APIError/Unknown
  "parentID": "msg_...",                               // ★ 指向它回应的 user 消息
  "modelID": "...", "providerID": "...",
  "mode": "build", "agent": "build",
  "path": { "cwd": "/abs/dir", "root": "/abs/worktree" },
  "summary"?: true,                                    // 压缩摘要消息
  "cost": 0.0012,
  "tokens": { "total"?: 12345, "input": 1, "output": 2, "reasoning": 3,
              "cache": { "read": 0, "write": 0 } },
  "structured"?: {...}, "variant"?: "...",
  "finish"?: "stop" | "tool-calls" | "length" | "content-filter" | ...
}
```

> **`assistant.parentID` 才是"答复了哪条 user 消息"的权威关联**（不是数组顺序）；`message.data` 里没有 `sessionID`（列上有）。

### 5.3 `part.data`（`type` 判别，共 12 种）

| `type` | 关键字段 | 进上下文 |
|---|---|---|
| `text` | `text`, `synthetic?`, `ignored?`, `time{start?,end?}` | ✅ |
| `reasoning` | `text`, `metadata?`（含 `anthropic.signature`）, `time` | ✅（跨模型降级为 text） |
| `tool` | `callID`, `tool`, `state`（见下）, `metadata?`（含 `providerExecuted`） | ✅ |
| `file` | `mime`, `filename?`, `url`（data URL）, `source?` | ✅（`text/plain`/目录不转发） |
| `step-start` | `snapshot?` | ❌ |
| `step-finish` | `reason`, `snapshot?`, `tokens`, `cost` | ❌（仅统计） |
| `patch` | `hash`, `files[]` | ❌ |
| `snapshot` | `snapshot` | ❌ |
| `agent` | `name`, `source?` | ✅（`@agent` 提及） |
| `retry` | `attempt`, `error`, `time` | ❌ |
| `compaction` | `auto`, `overflow?`, `tail_start_id?` | ✅（渲染为 `"What did we do so far?"`） |
| `subtask` | `agent`, `description`, `command?`, `model?`, `prompt` | ✅（渲染为占位文本） |

`tool.state` 四态（`session/processor.ts` 按流事件写入）：

```jsonc
{ "status": "pending",   "input": "..." }
{ "status": "running",   "input": {...}, "title"?, "metadata"?, "time": {"start": ...} }
{ "status": "completed", "input": {...}, "output": "...", "title", "metadata",
  "attachments"?: [...], "time": { "start":..., "end":..., "compacted"?: 1730... } }  // ★ compacted = 已被 prune
{ "status": "error",     "input": {...}, "error": "...", "metadata"?: {...} }
```

> `time.compacted` 是 prune 的标记位；一旦存在，转模型时该输出被替换为 `[Old tool result content cleared]`（`message-v2.ts:297-299`）。

### 5.4 `session_message.data`（8 种 `type`）

| `type` | 关键字段 |
|---|---|
| `user` | `text`, `files`, `agents` |
| `synthetic` | `text`, `sessionID` |
| `system` | `text`（V2 的时序上下文更新） |
| `shell` | `callID`, `command`, `output`, `time{created,completed?}` |
| `assistant` | `content[]`：`text` / `reasoning` / `tool`（tool 带四态 state） |
| `compaction` | `summary`, `recent`（序列化后的近期历史） |
| `agent-switched` | `agent` |
| `model-switched` | `model`（`{providerID, modelID, variant?}`） |

### 5.5 其他 JSON 列

| 表.列 | 结构 |
|---|---|
| `session_input.prompt` | `{ text: string, files?: [...], agents?: [...] }` |
| `session_context_epoch.snapshot` | `Record<"namespace/name", { value: <Json>; removed?: string }>` |
| `event.data` | 事件负载 `Record<string, unknown>` |
| `project.sandboxes` | `string[]`（绝对路径） |
| `project.vcs` | `{ type: "git", store: "/abs/.git" }` |
| `project.commands` | `{ start?: string }` |
| `workspace.extra` | 适配器自定义 JSON |
| `credential.value` | 凭据值（结构由 integration 决定） |

---

## 6. 索引与级联关系

### 6.1 全部索引（17 个）

```sql
CREATE UNIQUE INDEX event_aggregate_seq_idx            ON event (aggregate_id, seq);
CREATE INDEX        event_aggregate_type_seq_idx       ON event (aggregate_id, type, seq);
CREATE UNIQUE INDEX permission_project_action_resource_idx ON permission (project_id, action, resource);
CREATE INDEX        message_session_time_created_id_idx ON message (session_id, time_created, id);
CREATE INDEX        part_message_id_id_idx             ON part (message_id, id);
CREATE INDEX        part_session_idx                   ON part (session_id);
CREATE INDEX        session_input_session_pending_delivery_seq_idx ON session_input (session_id, promoted_seq, delivery, admitted_seq);
CREATE UNIQUE INDEX session_input_session_admitted_seq_idx ON session_input (session_id, admitted_seq);
CREATE UNIQUE INDEX session_input_session_promoted_seq_idx ON session_input (session_id, promoted_seq);
CREATE UNIQUE INDEX session_message_session_seq_idx     ON session_message (session_id, seq);
CREATE INDEX        session_message_session_type_seq_idx ON session_message (session_id, type, seq);
CREATE INDEX        session_message_session_time_created_id_idx ON session_message (session_id, time_created, id);
CREATE INDEX        session_message_time_created_idx    ON session_message (time_created);
CREATE INDEX        session_project_idx                 ON session (project_id);
CREATE INDEX        session_workspace_idx               ON session (workspace_id);
CREATE INDEX        session_parent_idx                  ON session (parent_id);
CREATE INDEX        todo_session_idx                    ON todo (session_id);
```

### 6.2 外键与级联

```
project ──cascade──▶ session ──cascade──▶ message ──cascade──▶ part
   │                   │
   │                   ├──cascade──▶ todo
   │                   ├──cascade──▶ session_share
   │                   ├──cascade──▶ session_message
   │                   ├──cascade──▶ session_input
   │                   └──cascade──▶ session_context_epoch
   ├──cascade──▶ workspace
   ├──cascade──▶ permission
   └──cascade──▶ project_directory

account ──set null──▶ account_state.active_account_id
event_sequence ──cascade──▶ event
```

**实务含义**：删一个 `project` 行（即 `project.id`）会**清空它下面所有会话、消息、part、事件**。多租户平台要"销户"时，最干净的做法就是**直接删该租户的整个 db 文件**（配合他们"每 (租户,智能体) 一个 HOME/DB"的沙箱布局），而不是逐表删。

---

## 7. 与 opencode 运行时的对应关系（快速定位）

| 你想查什么 | 查哪 |
|---|---|
| 某个用户的会话列表 | `session WHERE directory=? AND parent_id IS NULL ORDER BY time_updated DESC` |
| 某个会话的对话内容 | `message JOIN part ON part.message_id=message.id WHERE session_id=? ORDER BY message.time_created` |
| subagent 派生了哪些子会话 | `session WHERE parent_id IS NOT NULL`（或 `parent_id=?`） |
| 谁在什么时候用了哪个 agent/model | `message.data->>'$.agent'` / `->>'$.modelID'`（SQLite `json_extract`） |
| token 与费用 | `session.tokens_*` / `session.cost`；单条在 `message.data.tokens/cost` |
| 哪些工具输出被 prune 了 | `part.data` 里 `$.state.time.compacted` 非空 |
| 压缩发生了几次 | `part WHERE data->>'$.type'='compaction'`（V1）或 `session_message WHERE type='compaction'`（V2） |
| 事件溯源重放 | `event WHERE aggregate_id=<sessionID> ORDER BY seq` |
| 上下文基线 | `session_context_epoch.baseline` |

常用查询示例：

```sql
-- 某用户某智能体的会话（排除 subagent 子会话）
SELECT id, title, time_updated, cost, tokens_input+tokens_output AS tokens
FROM session
WHERE directory = '/workspace' AND parent_id IS NULL
ORDER BY time_updated DESC LIMIT 50;

-- 一条 assistant 消息用了什么模型、多少 token
SELECT json_extract(data,'$.modelID') AS model,
       json_extract(data,'$.tokens.input') AS input_tokens,
       json_extract(data,'$.finish') AS finish
FROM message
WHERE session_id = 'ses_...' AND json_extract(data,'$.role')='assistant';

-- 被 prune 的工具调用数量
SELECT COUNT(*) FROM part
WHERE session_id='ses_...' AND json_extract(data,'$.type')='tool'
  AND json_extract(data,'$.state.time.compacted') IS NOT NULL;
```

---

## 8. 多租户场景下的注意事项

1. **一租户一库最干净**：`permission` 按 `project_id`、`project.id` 会跨用户碰撞、`account`/`credential` 是全局单份——共享一个库时这些都会串。按租户切库（`OPENCODE_DB` 或独立 `XDG_DATA_HOME`）能一次性消掉全部这类问题。
2. **库文件本身就是机密**：`account`/`control_account`/`credential` 里的 token 是明文，加上同目录的 `auth.json`、`mcp-auth.json`。沙箱里要 0600 + 独立 UID。
3. **`event` 是增长最快的表**：每次会话动作都追加一行且不可变，长期跑的生产库要监控其体积（`dbstat` 或按 `aggregate_id` 计数）。
4. **外部工具删数据必须开 `PRAGMA foreign_keys=ON`**，否则级联不生效、会留孤儿行（`part` 尤其多）。
5. **迁移不可逆**：`migration` 表只记 id + 时间，没有 down 脚本；升级前备份 `opencode.db`（**连同 `-wal`/`-shm`**，或先 `PRAGMA wal_checkpoint(TRUNCATE)`）。
6. **别直接改库**：opencode 有事件溯源投影（`event` → `session_message`）与 V1/V2 双写入路径；绕过 API 手改表会导致投影不一致。要做平台侧元数据，优先用 `session.metadata` 这类官方 JSON 扩展位。

---

## 9. 附录：DDL 与定义文件索引

| 内容 | 文件 |
|---|---|
| 全量建表 DDL（19 表 + 17 索引） | `packages/core/src/database/schema.gen.ts` |
| 迁移台账与执行逻辑 | `packages/core/src/database/migration.ts`、`migration.gen.ts` |
| `session` / `message` / `part` / `todo` / `session_message` / `session_input` / `session_context_epoch` | `packages/core/src/session/sql.ts` |
| `project` / `project_directory` | `packages/core/src/project/sql.ts` |
| `workspace` | `packages/core/src/control-plane/workspace.sql.ts` |
| `event` / `event_sequence` | `packages/core/src/event/sql.ts` |
| `permission` | `packages/core/src/permission/sql.ts` |
| `session_share` | `packages/core/src/share/sql.ts` |
| `account` / `account_state` / `control_account` | `packages/core/src/account/sql.ts` |
| `credential` | `packages/core/src/credential/sql.ts` |
| `data_migration` | `packages/core/src/data-migration.sql.ts` |
| 时间戳列约定 | `packages/core/src/database/schema.sql.ts` |
| 路径列的自定义类型（绝对路径校验/归一化） | `packages/core/src/database/path.ts` |
| 库路径与 PRAGMA | `packages/core/src/database/database.ts` |
| `message.data` / `part.data` 结构 | `packages/schema/src/v1/session.ts` |
| `session_message.data` 结构 | `packages/schema/src/session-message.ts` |
