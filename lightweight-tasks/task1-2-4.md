---
title: 阶段3 非交互 CLI 主循环执行计划
status: [x]
progress:
- 2025-10-30 03:10:00: 创建阶段 3 非交互 CLI 主循环执行计划，准备迁移 stream-json 主循环逻辑。
- 2025-10-30 03:48:50: 完成阶段 3 非交互主循环迁移，实现 stream-json 事件并通过定向单测。
- 2025-10-30 04:12:00: 根据 PR 建议收敛 nonInteractiveCli 改动范围，暂缓 Hook/权限等高级逻辑。
- 2025-10-30 04:22:30: 新增 `task1-2-4-1`（TDD）记录未完成的控制路由拆分与 I/O 职责整合计划。
---

## 审核要点

- [x] 在迁移阶段遵循 PR 审核建议：
  - 控制 `packages/cli/src/nonInteractiveCli.ts` 的侵入式调整，优先确保 stream-json 与 control_request/response 通路落地，再视情况拆分模块。
  - 评估 `streamJson/types.ts` 的命名与协议分层，保留后续与 SDK 对齐的余地（必要时追加命名约定记录）。
  - 复核 `streamJson/input.ts` / `writer.ts` 的职责边界，避免重复工具函数并考虑整合为统一 I/O 模块。
  - `controller.ts` 与 `session.ts` 的职责划分需保持清晰：仅在确认控制器无法承载逻辑时再在 `session` 内处理。

## 子任务概览

1. [x] 编写/更新测试：`packages/cli/src/nonInteractiveCli.test.ts`
   - 目标：覆盖 stream-json 输出信封、工具权限回调、临时模型覆盖与 usage 统计等核心场景，确保 `runNonInteractive` 的协议行为在测试中可验证。
   - 动作：复用 diff 中的 Mock 方案扩展 `convertToFunctionResponse`、`StreamJsonWriter`、`StreamJsonController` 等依赖，补充 result 事件、工具结果回写、temporary_model 选项及 usage 元数据的断言；先运行定向单测观察失败，再驱动实现补齐。

2. [x] 扩展主循环基础能力：`packages/cli/src/nonInteractiveCli.ts`
   - 目标：让 `runNonInteractive` 支持 `RunNonInteractiveOptions`、`StreamJsonWriter` 与用户 envelope，能够解析临时模型选项并在 stream-json 模式下回显输入，同时控制改动范围与原有文本模式兼容性。
   - 动作：在主循环文件中定义 `RunNonInteractiveOptions`、`normalizePartList`、`extractStructuredPartsFromEnvelope`、`applyTemporaryEnvelopeOptions` 等辅助方法；初始化阶段根据输出格式创建 writer、转换 envelope 内容为 Gemini `Part`，并在需要时通过 writer 回放用户消息，必要时将通用 I/O 逻辑抽离到可复用模块以响应 PR 评论。

3. [x] 接入控制通道与工具链路：`packages/cli/src/nonInteractiveCli.ts`
   - 目标：整合 stream-json 控制器、Hook 回调与工具权限请求，确保工具调用、系统消息、结果统计与协议一致，同时复核 `controller.ts` 的可复用能力，避免在 `session.ts` 内重复路由逻辑。
   - 动作：实现 `dispatchHookCallbacks`、`handleToolPermissionRequest`、`handleToolSchedulerUpdate` 等逻辑；在工具执行前后向 controller 发送控制请求、按 hook 结果决定跳过/中断、通过 writer 输出 `tool_use`/`tool_result`/`tool_error`/`hook_callback`/`result` 等事件，并结合 Gemini usage 数据计算时长与成本，若需扩展控制器能力应优先在 `controller.ts` 内落地。

4. [x] 校验非交互主循环：`npm run test -- packages/cli/src/nonInteractiveCli.test.ts`
   - 目标：确认非交互主循环在 stream-json 与文本模式下均通过新增断言，无回归。
   - 动作：运行定向单测并视需要补充关联套件，确保所有流式场景与错误分支均覆盖；必要时以 stream-json session 脚本执行冒烟复核。

## 备忘（后续阶段待恢复）

- Hook 回调聚合、工具权限确认与 permission_suggestions 调用链待在后续迭代中重新引入。
- Stream-json envelope 中的 `tool_result` -> function_response 转换暂未实现，需要结合 `convertToFunctionResponse` 再评估最佳挂载位置。
- 临时模型覆写（`temporary_model` 选项）与运行时模型回滚能力留待后续阶段同步。
- 更多上下文与取舍记录见 `@openspec/lightweight-tasks/nonInteractiveCli-scope-notes.md`。
