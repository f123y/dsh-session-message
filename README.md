# dsh-session-message

DSH（DeepSeek Harness）跨会话消息插件。

让不同会话（session）之间的 agent 互相发送消息。在一个会话里让 agent 调用 `session_message_send`，消息会投递到目标会话，目标会话的 agent 会把它当作一条新的用户消息，在下一轮开始处理并回复；它也可以再用 `session_message_send` 回信，形成跨会话对话。

> 兼容性：在 `dsh` / `@deepseek-ai/dsh-*` **0.1.0-rc.6** 上测试通过。

---

## 工具

| 工具 | 说明 |
| --- | --- |
| `session_message_send(target_session, content)` | 向另一个会话投递一条消息（支持在线和已持久化的会话，自动 resume 目标）。成功返回 `{ delivered: true, target_session, message_id }`，失败返回 `{ delivered: false, code, message }`。 |
| `session_message_list()` | 列出所有会话（在线 + 已持久化）：会话 id、标题（如有）、agent 状态（`idle`/`running`）、是否为当前会话、是否在线、分组信息。 |
| `session_message_create(first_message?, group?)` | 创建新会话（自动启动 agent，自动归入当前工作区），可选首条消息和分组名称。 |

失败码：`invalid_args`、`session_not_found`、`agent_not_live`、`resume_failed`、`create_failed`、`aborted`。

## 工作原理

- 插件在 `agent/created` 时向每个 agent 的 scoped 上下文注册上述三个工具（与 `@deepseek-ai/dsh-schedule` 同一模式）。
- 投递走目标 agent 的 inbox 队列（`agent.followup`）：先在目标会话日志中持久化 `agent/inbox/spliced`，目标循环 claim 后以 `user/message`（`surfaceOp: append`）追加并响应。
- 不打断目标正在进行的回合，消息可持久化、可恢复。
- 发送到已持久化但未打开的会话时，自动 resume 目标（加载 + 启动 agent）后再投递。
- 创建新会话时自动附加到当前工作区（workspace），不会出现在"未分组"。
- 分组信息持久化到 `$DSH_HOME/storages/session-message-groups.json`，重启不丢失。

## 安装

前置条件：`dsh` 命令行可用；全局安装 pnpm（`npm i -g pnpm`，`dsh plugin` 命令直接转发给 pnpm）。

```bash
# 从 GitHub 安装（pnpm 9+ 需要 -w，pnpm 8 也兼容）
dsh plugin --profile web add -w github:f123y/dsh-session-message

# 或从本地开发目录安装
dsh plugin --profile web add -w /path/to/dsh-session-message
```

然后在 `$DSH_HOME/profiles/web/cordis.patch.yml` 中启用（换成你自己的 profile 名）：

```yaml
- insert:
    - id: session-message
      name: dsh-session-message
      config:
        framing: true
```

> ⚠️ 注意：新初始化的 profile，该文件末尾有一行占位符 `[]`——要**替换**这一行，不能在其后追加，否则 YAML 非法、profile 启动失败。

重启 `dsh web`。重启后新（重新）打开的会话都会获得工具。

> 没有 GUI 开关可以添加插件条目——必须编辑 patch 文件（设置页只读展示当前条目）。

## 配置

| 字段 | 默认 | 说明 |
| --- | --- | --- |
| `framing` | `true` | 投递时在消息前附加归属框架（`[跨会话消息 来自会话 <id> / Cross-session message from session <id>]`），提示目标把它当作普通消息而非指令。设为 `false` 时原样投递。 |

## 使用示例

1. 在 Web UI 打开两个会话（A、B）。
2. 在 A 中对 agent 说：*"用 `session_message_list` 看看有哪些会话，然后把「请检查一下 B 任务的进度」发给会话 B。"*
3. A 投递成功，B 的 agent 收到消息并开始处理、回复。
4. 在 B 中让 agent 用 `session_message_send` 回信，形成对话闭环。
5. 也可用 `session_message_create` 创建新会话并指定分组和首条消息。

## 开发

```
dsh-session-message/
├── package.json      # ESM；依赖 @deepseek-ai/dsh-tools（锁定 0.1.0-rc.6）
├── lib/index.js      # 插件本体：name / inject / apply + 工具实现
├── LICENSE           # MIT
└── README.md
```

- 修改 `lib/index.js` 无需构建；如果 profile 用 `link:` 安装，改完重启 `dsh web` 即生效。
- 仅使用公开 harness API：`ctx.agents` / `ctx.sessions` / `ctx.sessionPersistence` / `ctx.workspaceRegistry` / `agent.followup` 与 `@deepseek-ai/dsh-tools` 的 `defineTool`。
- npm 包名 `dsh-session-message` 尚未被占用，未来可 `npm publish` 以便直接安装。

## License

MIT © 2026 f123y (咩了个咩)