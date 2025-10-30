---
title: 阶段5 控制器与会话驱动（TDD 执行表）
status: [x]
progress:
  - 2025-10-30 13:40:00: 起草阶段 5 控制器与会话驱动的 TDD 计划。
  - 2025-10-30 15:45:00: 完成会话单测补全、实现入口联通并通过定向测试验证。
---

## 子任务概览

1. [x] 编写/更新测试：`packages/cli/src/streamJson/session.test.ts`  
   - 目标：覆盖会话循环的核心路径，包括初始化响应、用户消息入队、控制请求转发、工具结果回放以及 `--include-partial-messages` 触发 `stream_event` 的场景。  
   - 动作：
     1. 创建或扩展 `session.test.ts`，为 `initialize`→`control_response`、`user` 消息→`promptQueue`、`interrupt` 等事件写出失败断言。
     2. 模拟 `StreamJsonWriter`/`StreamJsonController` 行为，验证会话循环对 `runNonInteractive` 的调用次数、参数透传与增量事件输出。
     3. 运行定向测试（例如 `npx vitest run packages/cli/src/streamJson/session.test.ts`），确保在实现前观察到预期失败。

2. [x] 实现会话驱动：`packages/cli/src/streamJson/session.ts`  
   - 目标：串联输入解析、Prompt 队列、控制请求处理与 `runNonInteractive` 调用，确保启用 `--include-partial-messages` 时能通过 `StreamJsonWriter` 输出 `stream_event` 增量片段。  
   - 动作：
     1. 实现 `runStreamJsonSession` 主循环：读取解析的信封、排队 `user` 提示、委派控制请求给 `StreamJsonController`，并按需启动/中断 `runNonInteractive`。
     2. 维护 `AbortController`/`promptQueue`/`toolCallRegistry` 等上下文，确保长驻流程可恢复、终止或清理资源。
     3. 补齐 `StreamJsonController` 可能缺失的接口实现（如 `setActiveRunAbortController`）以匹配测试断言，保持接口最小化。

3. [x] 入口集成：`packages/cli/src/gemini.tsx` 等  
   - 目标：在会话能力就绪后，更新 CLI 顶层入口以根据参数切换至 stream-json 模式，同时维持现有文本流程不变。  
   - 动作：
     1. 为 `gemini.tsx` 撰写或扩展单测（如 `gemini.test.tsx`），覆盖 `--input-format stream-json` 与默认文本模式的分支。
     2. 实现入口分流逻辑，处理空 prompt、会话结束与退出码，确保 `runStreamJsonSession` 得到 `settings`/`config` 所需依赖。
     3. 运行相关测试验证入口切换不影响既有行为。

4. [x] 验证与冒烟  
   - 目标：在实现全部落地后复跑单测，并通过脚本验证基础场景能稳定输出。  
   - 动作：
     1. 运行 `npx vitest run packages/cli/src/streamJson/session.test.ts` 与入口相关测试，确认新增断言全部通过。
     2. 使用 `timeout 10 npm run stream-json-session -- --include-partial-messages` 搭配示例请求进行手动冒烟，检查 stdout 中的 `user`/`assistant`/`stream_event` 是否符合预期。

## 不包含的内容

- Hook 回调聚合、权限审批、MCP 控制请求等高级能力将留待后续阶段（6/7/8）补齐。
- 不扩展 `runNonInteractive` 内部逻辑，仅在会话与入口层协调已有接口，确保变更最小化。
