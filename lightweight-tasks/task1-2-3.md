---
title: 阶段2 Writer 能力执行计划
status: [x]
progress:
- 2025-10-30 01:55:00: 创建阶段 2 Writer 能力执行计划，准备按 TDD 推进。
- 2025-10-30 02:38:31: 标记阶段 2 Writer 任务开始，准备补齐测试与实现。
- 2025-10-30 02:42:51: 完成 Writer 能力阶段任务，单测通过并输出结构化事件。
---

## 子任务概览

1. [x] 编写/更新测试：`packages/cli/src/streamJson/writer.test.ts`
   - 目标：通过新增结果汇总、思维增量、工具调用与系统事件等断言，让 `StreamJsonWriter` 的核心行为在测试中全量曝光，确保回归时能精准定位协议变动。
   - 动作：复用 diff 中的 Mock 方案拦截 `process.stdout.write`，补全 `emitResult`、`createAssistantBuilder`、`emitSystemMessage` 等场景；先运行单测确认失败，再以实现驱动通过。

2. [x] 落地协议类型：`packages/cli/src/streamJson/types.ts`
   - 目标：定义 `StreamJsonContentBlock`、`StreamJsonOutputEnvelope`、`StreamJsonMessageStreamEvent` 等结构，并提供 `serializeStreamJsonEnvelope` 与解析错误类型，为 Writer 输出与输入校验打下基础。
   - 动作：参考 diff 中的字段命名与可选属性，补齐工具结果、控制通道和 `stream_event` 的联合类型，实现场景化的 JSON 解析异常抛出。

3. [x] 实现 Writer 与增量事件：`packages/cli/src/streamJson/writer.ts`
   - 目标：完成 `StreamJsonWriter` 及 `StreamJsonAssistantMessageBuilder`，支持结果统计写出、系统消息附带会话信息、工具结果回传、按需输出 `stream_event`（`message_start`/`content_block_delta` 等）、以及文本/思维块拼接。
   - 动作：实现 `partsToString`、`toolResultContent` 等辅助函数；在开启 `includePartialMessages` 时为每个增量事件生成 UUID 与 `message_id`，确保测试中可观察到 `thinking_delta` 与 `input_json_delta`；关闭该标志时则保持静默。

4. [x] 校验输出链路：`npm run test -- packages/cli/src/streamJson/writer.test.ts`
   - 目标：确认 Writer 与类型实现满足全部断言，特别是结果事件字段、增量事件过滤逻辑与工具结果序列化。
   - 动作：以新增测试为入口运行定向单测，随后在阶段收尾前执行一次相关包的聚合测试，确保无额外回归。
