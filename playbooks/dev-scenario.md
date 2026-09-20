# 场景：软件开发项目组（架构 → 实现 → 评审 → 测试）

前置：先把 [pm-template.md](pm-template.md) 发给 PM 会话，然后只发一行 goal。
下面是一个可直接套用的实例。

## Goal 示例

```text
给 dsh-session-message 插件加一个 session_message_broadcast 工具：
向指定 group 的全部在线会话群发同一条消息，返回逐个投递结果。
```

## PM 拆解示例（它应该产出类似这样的清单）

| 编号 | 任务 | 指派 | 完成标准 |
| --- | --- | --- | --- |
| T1 | 读插件源码，产出工具设计：参数、输出 schema、边界情况 | 架构-A1 | 设计文档一份，含 schema 草案 |
| T2 | 评审 T1 设计，指出问题 | 评审-R1 | 逐条意见（ blocker / 建议 分级） |
| T3 | 按定稿设计实现 + 语法/加载自测 | 实现-D1 | `node --check` 与模块 import 通过，diff 可读 |
| T4 | 按 T3 的 diff 写并跑冒烟验证 | 测试-T1 | 结果清单，全绿或列出失败项 |
| T5 | 终审 diff 与 README 更新 | 评审-R1 | 通过 / 打回 |

依赖链：T1 → T2 → T3 → T4 → T5，串行；如果功能更大，实现可拆 D1/D2 并行（按文件分工）。

## PM 建组示例（每条 first_message = 工人守则 + 职责）

- `session_message_create(title: "架构-A1", group: "dev-broadcast", first_message: "【工人守则全文】\n你的职责：负责设计。你会收到任务编号、背景和完成标准，产出设计文档并按守则回传。")`
- 同理建 `实现-D1`、`评审-R1`、`测试-T1`。

## PM 派工示例（T1）

```text
【任务 T1】
背景：本插件源码在 E:\d1\dsh-session-message（lib/index.js 是全部实现）。
要做什么：设计 session_message_broadcast 工具——参数（group、content、priority?）、
输出 schema（逐会话 delivered 结果 + 汇总）、对离线会话的处理策略。
完成标准：一份设计说明，含可直接抄进代码的 schema JSON。
完成后用 session_message_send 把结果发给本条消息开头框架行里的会话 id，
不要只在本会话里回复。
```

## PM 验收与插队示例

- 验收：收到 A1 回执后，一行记「T1 ✅ 设计已交」，立刻把设计转给 R1（T2）。
- 用户插话「broadcast 优先，先别管文档」→ PM 对在排队的任务用
  `session_message_queue(action: "promote", message_id: …)` 提级，或对后续派工用
  `priority: "immediate"`。
- R1 打回 T1 → PM 记「T1 ⛔ 打回：schema 缺 failure 汇总」，把意见转回 A1 重做。

## 交付

T5 通过后 PM 向用户交报告：diff 摘要、测试结论、遗留问题（如「文档未更新」）。
