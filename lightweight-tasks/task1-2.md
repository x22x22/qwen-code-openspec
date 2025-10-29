---
title: stream-json 迁移实施计划
status: [ ]
progress:
- 2025-10-30 00:12:42: 基于 task1-1.md 输出按阶段细化的 TDD 迁移步骤。
---

## 子任务概览

1. [x] 阶段 0：基础支撑  
   - `.gitignore` 增加 Python 缓存忽略；`scripts/build_package.js` 调整为宽容式 `tsc --build`；`package.json` 补充 `qwen` 与 `stream-json-session` 脚本，保障后续测试命令可直接执行。
2. [x] 阶段 1：参数解析与核心配置  
   - 先补充 `packages/cli/src/config/config.test.ts` 中的新用例，再更新 CLI 与核心配置文件，使 `--input-format/--output-format/--include-partial-messages` 得以正确解析、校验并传递至核心层，同时扩展 `nonInteractiveToolExecutor` 的回调接口。
3. [ ] 阶段 2：Writer 能力  
   - 以 `packages/cli/src/streamJson/writer.test.ts` 为驱动，实现基础类型定义与 `StreamJsonWriter`，确保结果事件、增量片段、工具结果等结构化输出全部符合预期。
4. [ ] 阶段 3：输入解析  
   - 根据 `packages/cli/src/streamJson/input.test.ts` 的断言，实现 `StreamJson` 输入解析器，覆盖初始化响应、用户消息拼接及无消息报错场景，为会话循环提供稳定输入面。
5. [ ] 阶段 4：控制器与会话驱动  
   - 先落地 `packages/cli/src/streamJson/session.test.ts`，随后实现控制器与会话主循环，串联队列调度、控制请求/MCP 注册与 `runNonInteractive` 调用链。
6. [ ] 阶段 5：非交互 CLI 主循环  
   - 依照 `packages/cli/src/nonInteractiveCli.test.ts` 中新增测试改造 `nonInteractiveCli.ts`，将 stream-json writer、控制器、工具回执桥接进原有流程，并确保传统文本模式不受影响。
7. [ ] 阶段 6：入口切换与顶层集成  
   - 更新 `packages/cli/src/gemini.tsx`，在配置为 stream-json 模式时进入 `runStreamJsonSession`，处理常驻与空 prompt 等边界情况，必要时补充额外测试。
8. [ ] 阶段 7：文档与示例  
   - 最后迁入 `docs/cli/index.md`、`docs/examples/stream-json` 及相关 RFC，依据 README 指引执行脚本与日志对比，核实协议文档与实现一致。
