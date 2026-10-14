# PrivateClaude 设计文档

## 1. 项目定位

PrivateClaude 是从零实现的本地 Agent 运行时（mini Claude Code）：

- 常驻 daemon 承载全部运行状态（会话、工具执行、权限审批、事件落盘），客户端只做展示与交互
- 流式 LLM 对话 + 工具调用循环 + 权限审批 + 上下文治理 + 会话回放
- 单用户、本地、macOS / Linux

**非目标**：多租户 / 远程访问、工具沙箱隔离、Windows、浏览器前端（D16 已定 rich CLI）。

## 2. 总体架构

```
┌──────────────┐   NDJSON 行 / JSON-RPC 2.0   ┌─────────────────────────────┐
│   pc (CLI)   │ ◀──────── UDS socket ──────▶ │        pcd (daemon)         │
│ rich 渲染/交互│        ~/.private_claude/    │  ┌───────────────────────┐  │
└──────────────┘        daemon.sock           │  │ session   agent loop  │  │
        │                                     │  │ tools     permissions │  │
        │ 权限审批：daemon → client 的          │  │ events    compaction  │  │
        │ 反向 JSON-RPC request                │  │ memory    extensions  │  │
                                              │  └──────────┬────────────┘  │
                                              │  SQLite（事件 / 摘要 / 审批）│
                                              └─────────────────────────────┘
```

- **D1**：daemon + 多客户端。客户端重启不丢上下文；权限与事件集中落盘。
- **D2**：UDS 回环传输，socket 文件放在 `0700` 目录内，文件权限即访问控制。
- **D3**：NDJSON 线协议，可 grep、可 head，调试体验优先。
- **D4**：JSON-RPC 2.0 信封，只用所需子集（request / response / notification），不实现 batch。

## 3. 目录结构

```
src/private_claude/
├── cli/                 # pc：rich 轻量前端（D16）
│   ├── main.py          #   参数解析、连接失败提示
│   └── commands/        #   ping / chat / run / sessions / replay / perms
├── daemon/              # pcd
│   ├── server.py        #   UDS 监听、连接生命周期
│   └── app.py           #   组件组装（依赖注入入口）
├── protocol/            # daemon 与 cli 共用的契约（D3–D6）
│   ├── envelope.py      #   JSON-RPC 2.0 信封
│   ├── methods.py       #   method 名 + 参数/结果模型
│   └── events.py        #   事件通知模型（discriminated union）
├── transport/           # D2：UDS client / server，NDJSON 编解码
├── core/
│   ├── loop.py          # D7：agent 循环
│   ├── session/         #   会话状态与消息历史
│   ├── llm/             # D8：anthropic SDK 直连（流式 + prompt caching）
│   ├── compact/         # D9：上下文治理
│   ├── permissions/     # D10：policy / storage / manager 三层
│   ├── events/          # D11：EventBus（串行 await，见 §8）
│   ├── store/           # D12：SQLite 落盘
│   ├── tools/           # D13：registry + 内置工具（含 edit）
│   ├── mcp/             # D14：MCP client loader
│   ├── skills/          # D14：Skills loader
│   ├── subagents/       # D14：Subagents
│   └── memory/          # D15：会话摘要持久化
└── config.py            # D17：四级配置
```

与参考实现的本质差异：`protocol/` 与 `transport/` 提升为顶层共享包——契约是 daemon 与 cli 之间对等的东西，不属于任何一侧的内脏。

## 4. 线协议与消息契约（D3–D6）

### 4.1 信封

NDJSON，每行一个 JSON-RPC 2.0 对象：`{"jsonrpc":"2.0","id":..,"method":..,"params":..}`。

三类流量：

| 方向 | 形态 | 用途 |
|---|---|---|
| client → daemon | request | `ping` / `session.*` / `permission.respond` |
| daemon → client | notification | 事件流（method 统一为 `event`，params 是带 `type` 判别的事件模型） |
| daemon → client | request | 权限审批 `permission.request`（阻塞等待用户决定） |

### 4.2 消息模型（D5）

pydantic v2 + `type` 字段判别联合。所有跨进程消息都从这个模型集序列化/反序列化，两侧共用 `protocol/` 包。

### 4.3 协议文档（D6）

脚本从模型导出 JSON Schema（`docs/protocol/schema.json`），CI 中重新导出并 diff 防漂移。不手写线协议文档。

## 5. 核心运行时

### 5.1 Agent 循环（D7）

单循环 runner，每个 turn：

1. 取会话历史 + 系统提示 + 工具表，调 LLM（流式）
2. 流式增量通过事件总线广播给订阅的客户端
3. 响应含 `tool_use` → 权限检查（§5.3）→ 执行工具 → `tool_result` 回填历史
4. tool_result 落盘后检查是否触发 compaction（§5.2）
5. 有 tool_use 则回到 1，否则 turn 结束

### 5.2 上下文治理（D9）

- **触发点在 tool_result 回填之后**：轮边界是安全压缩点——Anthropic API 要求 `tool_use` / `tool_result` 严格配对，任意时刻压缩可能撕裂配对直接 400
- token budget：估算超过阈值（context window 的约 70%，可配）才触发
- 压缩动作：LLM 生成对话摘要 → 历史替换为 system + 摘要 + 最近 N 条
- 超大 `tool_result` 内容截断（保留头尾 + 省略标记），无论是否触发压缩都做

### 5.3 权限模型（D10）

三层，数据模型自设计：

- **policy**：规则求值。规则 = `(tool, pattern, effect)`，effect ∈ `allow / deny / ask`，pattern 按工具语义解释（bash 用前缀、文件工具用路径 glob）
- **manager**：决策编排——policy 命中即返回；`ask` 或无规则时向持有会话的客户端发 `permission.request`，带超时默认拒绝
- **storage**：审批结果持久化到 SQLite（`permission_rules` 表），重启后 remembered 决定仍生效

### 5.4 LLM 接入（D8）

anthropic SDK 直连：`AsyncAnthropic` + 流式；`cache_control` 打在 system 与工具表上。不做 provider 抽象——换模型不是本项目的目标场景。

### 5.5 事件分发（D11）

进程内 EventBus，订阅者按注册顺序**串行 await**。已知缺陷（一个慢订阅者阻塞其后所有订阅者）记录在案；本实现订阅者均为本地快 IO（SQLite 写入、客户端转发），选择保持简单。升级路径已规划：每订阅者独立队列，接口不变，仅改 `publish` 内部。

### 5.6 追踪落盘（D12）

SQLite（WAL 模式），路径默认 `~/.private_claude/trace.db`：

```sql
sessions(id, created_at, cwd, title)
runs(id, session_id, started_at, ended_at, status)
events(id, run_id, ts, type, payload_json)      -- append-only，回放的数据源
permission_rules(id, tool, pattern, effect, created_at)
summaries(id, session_id, created_at, content, token_count)  -- D15
```

回放 = 从 `events` 按时间序重建会话视图，不单独存消息副本。

### 5.7 记忆（D15）

会话摘要持久化到 SQLite：compaction 产生的摘要与turn 结束时的会话摘要写入 `summaries`；新会话启动时注入最近 K 条摘要作为背景上下文。不做 CLAUDE.md 类文件注入 loader。

## 6. 工具与扩展（D13–D14）

内置工具（registry 装饰器注册，参数与返回都是 pydantic 模型）：

| 工具 | 说明 |
|---|---|
| `read` / `write` / `list` / `grep` | 常规文件操作 |
| `edit` | **参考实现没有的工具**：`old_string` 全文件唯一匹配替换，失败时带行号与近邻上下文反馈 |
| `bash` | 超时 + 输出截断上限走配置默认值 |

扩展机制（D14，三个 loader 全保留）：MCP（stdio client，工具动态挂入 registry）、Skills（目录式技能包注入系统提示）、Subagents（声明式子代理 + 对应 tool）。

## 7. 前端与配置

### 7.1 CLI（D16）

rich 轻量 CLI，命令：`pc ping` / `pc chat`（交互式，流式渲染）/ `pc run "<prompt>"`（一次性）/ `pc sessions` / `pc replay <id>` / `pc perms`。权限审批在 chat 内联弹出确认。

### 7.2 配置（D17）

四级优先：内置默认 → `~/.private_claude/config.toml` → `.env` → 环境变量。字段：API key、model、socket 路径、DB 路径、compaction 阈值、bash 超时/输出上限。

## 8. 命名（D19）

包名 `private_claude`；命令 `pc`（CLI）、`pcd`（daemon，unix `*d` 后缀）；`pc-tui` 随 D16 取消。数据目录 `~/.private_claude/`。定了就全局统一。

## 9. 开放问题

| # | 问题 | 当前默认（不引入新决策，仅占位） |
|---|---|---|
| O1 | `pc` 发现 daemon 未启动时是否自动拉起 `pcd` | 不自动拉起，报错并提示手动运行 `pcd` |
| O2 | 多客户端并发写同一会话 | 单写者（创建者），其他客户端只读订阅事件流 |
| O3 | bash 超时 / 输出截断的具体默认值 | 实现时定，进 config 默认值并在 decisions.md 补一行 |
| O4 | prompt caching 的打点范围与收益验证 | 先 system + 工具表，落地后用 trace 里的 token 用量验证 |
