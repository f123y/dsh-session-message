# dsh-session-message

**DSH（DeepSeek Harness）跨会话消息插件 / Cross-session messaging plugin for DSH**

让不同会话（session）之间的 agent 互相发送消息。
Let agents in different sessions send messages to each other.

在一个会话里让 agent 调用 `session_message_send`，消息会投递到目标会话，目标会话的
agent 会把它当作一条新的用户消息，在下一轮开始处理并回复；它也可以再用
`session_message_send` 回信，形成跨会话对话。

Call `session_message_send` from one session and the message is delivered to the target
session — its agent receives it as a new user message, processes it in its next turn,
and can reply back with the same tool, forming a cross-session conversation.

> 兼容性 / Compatibility：在 `dsh` / `@deepseek-ai/dsh-*` **0.1.0-rc.6** 上测试通过。
> Tested with **0.1.0-rc.6**.

---

## 简体中文

### 工具 / Tools

| 工具 Tool | 说明 Description |
| --- | --- |
| `session_message_send(target_session, content)` | 向另一个会话投递一条消息（支持在线和已持久化的会话，自动 resume 目标）。Deliver a message to another session (supports live and persisted sessions; auto-resumes the target). |
| `session_message_list()` | 列出所有会话（在线 + 已持久化）：会话 id、标题（如有）、agent 状态（`idle`/`running`）、是否为当前会话、是否在线。List ALL sessions (live + persisted). |
| `session_message_create(first_message?)` | 创建新会话（自动启动 agent），可选首条消息。Create a new session with an agent, optionally with a first message. |

失败码为封闭集合 / Failure codes are a closed union：`invalid_args`、`session_not_found`、`agent_not_live`、`resume_failed`、`create_failed`、`aborted`。

### 工作原理 / How it works

- 插件在 `agent/created` 时向每个 agent 的 scoped 上下文注册上述两个工具（与 `@deepseek-ai/dsh-schedule` 同一模式）。On `agent/created`, the two tools are registered into each agent's scoped context (same pattern as `@deepseek-ai/dsh-schedule`).
- 投递走目标 agent 的 inbox 队列（`agent.followup`）：先在目标会话日志中持久化 `agent/inbox/spliced`，目标循环 claim 后以 `user/message`（`surfaceOp: append`）追加并响应。Delivery goes through the target agent's inbox (`agent.followup`): an `agent/inbox/spliced` event is durably appended first, then the target's loop appends it as a `user/message` before responding.
- 不打断目标正在进行的回合，消息可持久化、可恢复。Delivery never interrupts an in-flight turn, and messages survive restarts.
- **新功能**：发送到已持久化但未打开的会话时，自动 resume 目标（加载 + 启动 agent）后再投递。**New**: sending to a persisted (not open) session auto-resumes it before delivery.

### 安装 / Install

前置条件 / Prerequisites：`dsh` 命令行可用；全局安装 pnpm（`npm i -g pnpm`，`dsh plugin` 命令直接转发给 pnpm）。`dsh` CLI on PATH and `pnpm` installed globally — the `dsh plugin` command forwards to pnpm.

```bash
# 1. 从 GitHub 安装 / install from GitHub
#    （pnpm 9+ 需要在 workspace 根加 -w；pnpm 8 也兼容该参数）
#    (pnpm 9+ requires -w to add into the workspace root; harmless on pnpm 8)
dsh plugin --profile web add -w github:f123y/dsh-session-message

#    开发时也可用本地路径 / ...or from a local checkout
#    dsh plugin --profile web add -w /path/to/dsh-session-message
```

2. 在 `$DSH_HOME/profiles/web/cordis.patch.yml` 中启用（换成你自己的 profile 名）/
   Enable it in `$DSH_HOME/profiles/web/cordis.patch.yml` (replace `web` with your profile):

```yaml
- insert:
    - id: session-message
      name: dsh-session-message
      config:
        framing: true
```

   ⚠️ 注意 / Note：新初始化的 profile，该文件末尾有一行占位符 `[]`——要**替换**这一行，
   不能在其后追加，否则 YAML 非法、profile 启动失败。A freshly initialized profile's
   `cordis.patch.yml` ends with a placeholder line `[]` — **replace** that line; appending
   after it makes the file invalid YAML.

3. 重启 / Restart：`dsh web`。重启后新（重新）打开的会话都会获得工具；插件加载前已存在的 agent 不会补发。Agents created after the plugin loads get the tools; already-running sessions pick them up after a restart.

> 没有 GUI 开关可以添加插件条目——必须编辑 patch 文件（设置页只读展示当前条目）。
> There is no GUI switch to add a plugin entry — the patch file edit is required.

### 配置 / Config

| 字段 Field | 默认 Default | 说明 Description |
| --- | --- | --- |
| `framing` | `true` | 投递时在消息前附加中英双语归属框架（`[跨会话消息 来自会话 <id> / Cross-session message from session <id>]`），提示目标把它当作普通消息而非指令。设为 `false` 时原样投递。Prefix delivered messages with a bilingual attribution frame; set `false` to deliver verbatim. |

### 使用示例 / Usage example

1. 在 Web UI 打开两个会话（A、B）。Open two sessions (A and B).
2. 在 A 中对 agent 说：*"用 `session_message_list` 看看有哪些会话，然后把「请检查一下 B 任务的进度」发给会话 B。"* Ask A's agent to list sessions and send a message to B.
3. A 投递成功，B 的 agent 收到消息并开始处理、回复。A delivers; B receives and responds.
4. 在 B 中让 agent 用 `session_message_send` 回信，形成对话闭环。B can reply back with the same tool.
5. 也可用 `session_message_create` 创建一个新会话并自动投递首条消息。Use `session_message_create` to create a new session with a first message.

### 限制 / Limitations

- 发送到已持久化但未打开的会话时，会自动 resume 目标（加载 + 启动 agent），需等待几秒。Sending to a persisted (not open) session auto-resumes it — expect a few seconds delay.
- 允许发给自己（echo）。Sending to yourself is allowed.
- 工具描述、README 均为中英双语。Tool descriptions and README are bilingual (中文 + English).

### 开发 / Development

```
dsh-session-message/
├── package.json      # ESM；依赖 @deepseek-ai/dsh-tools（锁定 0.1.0-rc.6）
├── lib/index.js      # 插件本体：name / inject / apply + 工具实现
├── LICENSE           # MIT
└── README.md
```

- 修改 `lib/index.js` 无需构建；如果 profile 用 `link:` 安装，改完重启 `dsh web` 即生效。No build step; `link:` installs pick up changes on restart.
- 仅使用公开 harness API：`ctx.agents` / `ctx.sessions` / `ctx.sessionPersistence` / `agent.followup` 与 `@deepseek-ai/dsh-tools` 的 `defineTool`。Uses only public harness APIs.
- npm 包名 `dsh-session-message` 尚未被占用，未来可 `npm publish` 以便 `dsh plugin --profile web add dsh-session-message` 直接安装。The npm name is unclaimed — publishing later enables direct registry installs.

---

## English

### Tools

| Tool | Description |
| --- | --- |
| `session_message_send(target_session, content)` | Deliver a message to another session (supports live and persisted sessions; auto-resumes the target). |
| `session_message_list()` | List ALL sessions (live + persisted): id, title, status, current, live flag. |
| `session_message_create(first_message?)` | Create a new session with an agent, optionally with a first message. |

Failure codes are a closed union: `invalid_args`, `session_not_found`, `agent_not_live`, `resume_failed`, `create_failed`, `aborted`.

### How it works

- On `agent/created`, the plugin registers the two tools into each agent's scoped context (same pattern as `@deepseek-ai/dsh-schedule`).
- Delivery goes through the target agent's inbox (`agent.followup`): an `agent/inbox/spliced` event is durably appended to the target session's log; the target's loop claims the message and appends it as a `user/message` (`surfaceOp: append`) before responding.
- Delivery never interrupts the target's in-flight turn, and messages survive restarts.
- **New**: sending to a persisted (not open) session auto-resumes it before delivery.

### Install

Prerequisites: `dsh` CLI on PATH, and `pnpm` installed globally (`npm i -g pnpm`) — the `dsh plugin` command forwards to pnpm.

```bash
# 1. install from GitHub (pnpm 9+ requires -w to add into the workspace root;
#    the flag is harmless on pnpm 8)
dsh plugin --profile web add -w github:f123y/dsh-session-message

#    ...or from a local checkout while developing
dsh plugin --profile web add -w /path/to/dsh-session-message
```

2. Enable it in `$DSH_HOME/profiles/web/cordis.patch.yml` (replace `web` with your profile name if you use a different one):

```yaml
- insert:
    - id: session-message
      name: dsh-session-message
      config:
        framing: true
```

   Note: a freshly initialized profile's `cordis.patch.yml` ends with a placeholder line `[]` — **replace** that line with the insert entry above; appending after it makes the file invalid YAML and the profile will fail to boot.

3. Restart `dsh web`. Agents created after the plugin loads get the tools; already-running sessions pick them up after a restart.

There is no GUI switch to add a plugin entry — the patch file edit in step 2 is required (the settings page only lists the current entries). Verified end-to-end on a fresh profile (pnpm 9.1.4, Windows).

### Config

| Field | Default | Description |
| --- | --- | --- |
| `framing` | `true` | Prefix delivered messages with a bilingual attribution frame (`[跨会话消息 来自会话 <id> / Cross-session message from session <id>]`) plus a line telling the target to treat the content as a normal message, not instructions. Set `false` to deliver verbatim. |

### Usage example

1. Open two sessions (A and B) in the web UI.
2. In session A, ask the agent: *"Use `session_message_list` to see live sessions, then send '请检查一下 B 任务的进度' to session B."*
3. Session A's agent delivers the message; session B's agent receives it and responds.
4. In session B, ask the agent to reply via `session_message_send` — a conversation loop.
5. Use `session_message_create` to create a new session with a first message.

### Limitations

- **Sending to a persisted (not open) session auto-resumes it** — the target is loaded from disk, an agent is started, and the message is delivered. Expect a few seconds delay.
- Sending to yourself is allowed (echo).
- Tool descriptions and README are bilingual (中文 + English).

### Development

```
dsh-session-message/
├── package.json      # ESM; depends on @deepseek-ai/dsh-tools (pinned to 0.1.0-rc.6)
├── lib/index.js      # plugin: name / inject / apply + tool implementations
├── LICENSE           # MIT
└── README.md
```

- Edit `lib/index.js`; no build step. If your profile installs the plugin via `link:`, changes apply without re-installing — just restart `dsh web`.
- The plugin uses only public harness APIs: `ctx.agents` / `ctx.sessions` / `ctx.sessionPersistence` / `agent.followup` and `defineTool` from `@deepseek-ai/dsh-tools`.
- The npm package name `dsh-session-message` is unclaimed — `npm publish` can be added later so users can `dsh plugin --profile web add dsh-session-message`.

---

## License

MIT © 2026 f123y (咩了个咩)