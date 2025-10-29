---
title: 阶段1 参数解析与核心配置（TDD 执行表）
status: [x]
progress:
- 2025-10-30 01:36:51: 完成阶段 1，已补充参数解析测试、实现 CLI 配置透传，并扩展核心配置与工具执行接口。
---

## 子任务概览

1. [x] 编写/更新测试：`packages/cli/src/config/config.test.ts`  
   - 目标：新增 `--include-partial-messages` 依赖 `--output-format stream-json` 的报错用例，以及 stream-json 参数解析/透传的正向用例。  
   - 动作：整理断言覆盖 CLI 参数解析与 Config getter；运行针对性测试观察失败，作为实现前的预期输出。
2. [x] 实现 CLI 参数解析与配置写入：`packages/cli/src/config/config.ts`  
   - 目标：定义 `--input-format/--output-format/--include-partial-messages` 参数，约束非法组合，并在 `loadCliConfig` 中注入对应字段。  
   - 动作：依据测试失败定位补齐解析逻辑，更新交互/非交互模式判定，并再次运行测试确认解析层通过。
3. [x] 扩展核心配置与工具执行接口  
   - 目标：在 `packages/core/src/config/config.ts` 持久化输入/输出格式与增量标记，并在 `packages/core/src/core/nonInteractiveToolExecutor.ts` 增加回调选项，支撑后续 stream-json 流程。  
   - 动作：根据测试缺口增加字段、访问器与可选回调参数；执行相关测试验证核心层改动无回归。
