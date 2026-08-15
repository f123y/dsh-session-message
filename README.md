# dsh-session-message

DSH (DeepSeek Harness) plugin: let agents in **different sessions send messages to each other**.

Call `session_message_send` from one session and the message is delivered to the target
session — its agent receives it as a new user message, processes it in its next turn,
and can reply back with the same tool, forming a cross-session conversation.

Compatibility: tested with `dsh` / `@deepseek-ai/dsh-*` **0.1.0-rc.6**.

## Tools

| Tool | Description |
| --- | --- |
| `session_message_send(target_session, content)` | Deliver a message to another live session. The target agent picks it up in its next turn. Returns `{ delivered: true, target_session, message_id }` on success, or `{ delivered: false, code, message }` on failure. |
| `session_message_list()` | List all live sessions: `session_id`, optional title, agent status (`idle`/`running`), and whether it is the current session. Call this first to discover target ids. |

Failure codes are a closed union: `invalid_args`, `session_not_found`, `agent_not_live`, `aborted`.

## How it works

- On `agent/created`, the plugin registers the two tools into each agent's scoped context
  (same pattern as `@deepseek-ai/dsh-schedule`).
- Delivery goes through the target agent's inbox (`agent.followup`):
  1. an `agent/inbox/spliced` event is durably appended to the target session's log;
  2. the target's loop claims the message and appends it as a `user/message`
     (`surfaceOp: append`) before responding.
- So delivery never interrupts the target's in-flight turn, and messages survive restarts.

## Install

Prerequisites: `dsh` CLI on PATH, and `pnpm` installed globally (`npm i -g pnpm`)
— the `dsh plugin` command forwards to pnpm.

The plugin is a plain dependency of your profile (not a bundle), enabled through the
patch layer:

```bash
# 1. install from GitHub (pnpm 9+ requires -w to add into the workspace root;
#    the flag is harmless on pnpm 8)
dsh plugin --profile web add -w github:f123y/dsh-session-message

#    ...or from a local checkout while developing
dsh plugin --profile web add -w /path/to/dsh-session-message
```

2. Enable it in `$DSH_HOME/profiles/web/cordis.patch.yml` (replace `web` with your
   profile name if you use a different one):

```yaml
- insert:
    - id: session-message
      name: dsh-session-message
      config:
        framing: true
```

   Note: a freshly initialized profile's `cordis.patch.yml` ends with a placeholder
   line `[]` — **replace** that line with the insert entry above; appending after it
   makes the file invalid YAML and the profile will fail to boot.

3. Restart `dsh web`. Agents created after the plugin loads get the tools;
   already-running sessions pick them up after a restart.

There is no GUI switch to add a plugin entry — the patch file edit in step 2 is
required (the settings page only lists the current entries). Verified end-to-end on a
fresh profile (pnpm 9.1.4, Windows).

### Config

| Field | Default | Description |
| --- | --- | --- |
| `framing` | `true` | Prefix delivered messages with `[跨会话消息 来自会话 <id>]` plus a line telling the target to treat the content as a normal message, not instructions. Set `false` to deliver verbatim. |

## Usage example

1. Open two sessions (A and B) in the web UI.
2. In session A, ask the agent: *"Use `session_message_list` to see live sessions, then
   send '请检查一下 B 任务的进度' to session B."*
3. Session A's agent delivers the message; session B's agent receives it and responds.
4. In session B, ask the agent to reply via `session_message_send` — a conversation loop.

## Limitations

- **Target must be a live session**: only sessions currently open (with a running agent)
  can receive messages. Sending to a closed session returns `session_not_found` /
  `agent_not_live` — nothing is silently dropped.
- Sending to yourself is allowed (echo).
- Tool descriptions are in English (they are model-facing); UI/README are in Chinese.

## Development

```
dsh-session-message/
├── package.json      # ESM; depends on @deepseek-ai/dsh-tools (pinned to 0.1.0-rc.6)
├── lib/index.js      # plugin: name / inject / apply + tool implementations
├── LICENSE           # MIT
└── README.md
```

- Edit `lib/index.js`; no build step. If your profile installs the plugin via `link:`,
  changes apply without re-installing — just restart `dsh web`.
- The plugin uses only public harness APIs: `ctx.agents` / `ctx.sessions` /
  `agent.followup` and `defineTool` from `@deepseek-ai/dsh-tools`.
- The npm package name `dsh-session-message` is unclaimed — `npm publish` can be added
  later so users can `dsh plugin --profile web add dsh-session-message`.
