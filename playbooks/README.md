# Playbooks：多会话项目组编排模式

用 [dsh-session-message](../README.md) 把多个 DSH 会话组织成一支「项目组」：
**一个项目经理（PM）会话吃下 goal，自动建组、拆任务、派活、验收、催办**，
工人会话只管执行和回执。提示词就是全部——不需要任何额外编排代码。

```
                ┌──────────────┐
   goal ──────▶ │  PM 会话      │  拆解 / 派工 / 验收 / 巡检 / 汇报
                └──────┬───────┘
        session_message_send（排队 / immediate 插队 / promote 人工插队）
                       │
      ┌────────────────┼────────────────┐
      ▼                ▼                ▼
 ┌─────────┐     ┌─────────┐     ┌─────────┐
 │ 工人 A   │     │ 工人 B   │     │ 工人 C   │   执行 → session_message_send 回执
 └─────────┘     └─────────┘     └─────────┘
        （session_message_create 建立，group = 项目组名，title = 角色名）
```

## 快速上手（3 步）

1. 新建一个会话，把 [pm-template.md](pm-template.md) 的提示词整段发给它。
2. 再发你的 goal（一句话即可）。
3. 看它开工：它会建工人、派活、向你汇报验收进度。随时可以直接插话下指令——
   你是 PM 的老板。

## 文件

| 文件 | 内容 |
| --- | --- |
| [pm-template.md](pm-template.md) | PM 提示词模板（核心资产，内含工人守则）+ 设计注解 |
| [dev-scenario.md](dev-scenario.md) | 开发场景：架构 → 实现 → 评审 → 测试 的项目组实例 |
| [pentest-scenario.md](pentest-scenario.md) | 渗透场景（仅限授权测试/靶场）：侦察 → 分析 → 报告 的项目组实例 |

## 工具能力速查

| 能力 | 用法 |
| --- | --- |
| 建项目组 | `session_message_create`：`group` = 组名，`title` = 侧栏显示名（钉住），`first_message` = 工人守则 + 职责 |
| 派活 | `session_message_send`：普通排队；`priority: "immediate"` 急件插队 |
| 看积压 | `session_message_queue(target)`：`next_turn` 排队 / `next_step` 插队两条队列 + 状态 |
| 改队列 | `session_message_queue(action)`: `"promote"` 人工插队、`"remove"` 撤单、`"clear"` 清空 |
| 点名 | `session_message_list(query: 组名)`：谁在线、谁在忙 |
| 回执 | 工人用 `session_message_send` 发回框架行里的 PM 会话 id |

## 已知边界（写进模板里管理，而不是假装不存在）

- 插队不能打断工人**正在执行**的命令，只能让它下一步最先处理；
- 工人静默失联（中止/异常）不会有回执，PM 必须定期巡检；
- 所有会话由插件的插件实例持有，`patchReload: live` 的 profile 重载插件会重建会话组。
