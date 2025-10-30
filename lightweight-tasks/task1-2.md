---
title: stream-json 迁移实施计划
status: [-]
progress:
- 2025-10-30 00:12:42: 基于 task1-1.md 输出按阶段细化的 TDD 迁移步骤。
- 2025-10-30 02:50:32: 完成阶段 2 Writer 能力，落地类型定义、写出器与单测校验。
- 2025-10-30 04:22:30: 启动 task1-2-4-1（TDD），继续处理控制路由与 I/O 整合。
- 2025-10-30 13:35:00: 完成阶段 4 输入解析，串联 stream-json 输入解析器与单测验证。
---

## 铁律

1. 不可以因为本任务删除任何当前分支下的任何功能代码，包括相似的功能代码也不可以删除。
2. 要用最小代码量进行迁移，比如存在一些描述类、大小写类的内容可能存在更好的表述，但是也不要变更原有内容。
3. 代码移植不代表一定是文件级别的移植，如果代码文件存在冲突，就进行代码块的方式迁移。

## 子任务概览

1. [x] 阶段 0：基础支撑  
   - `.gitignore` 增加 Python 缓存忽略；`scripts/build_package.js` 调整为宽容式 `tsc --build`；`package.json` 补充 `qwen` 与 `stream-json-session` 脚本，保障后续测试命令可直接执行。
2. [x] 阶段 1：参数解析与核心配置  
   - 先补充 `packages/cli/src/config/config.test.ts` 中的新用例，再更新 CLI 与核心配置文件，使 `--input-format/--output-format/--include-partial-messages` 得以正确解析、校验并传递至核心层，同时扩展 `nonInteractiveToolExecutor` 的回调接口。
3. [x] 阶段 2：Writer 能力  
   - 以 `packages/cli/src/streamJson/writer.test.ts` 为驱动，实现基础类型定义与 `StreamJsonWriter`，确保结果事件、增量片段、工具结果等结构化输出全部符合预期。
4. [x] 阶段 3：非交互 CLI 主循环  
   - 依照 `packages/cli/src/nonInteractiveCli.test.ts` 中新增测试改造 `nonInteractiveCli.ts`，把 stream-json writer、工具回执与统计写出融入现有循环，为最小化端到端输出打好基础。
5. [x] 阶段 4：输入解析  
   - 根据 `packages/cli/src/streamJson/input.test.ts` 的断言，实现 `stream-json` 输入解析器，覆盖初始化响应、用户消息拼接及无消息报错，保证 CLI 在管道模式下能及时获取事件。
6. [ ] 阶段 5：控制器与会话驱动 + 入口切换  
   - 目标：以 `packages/cli/src/streamJson/session.test.ts` 为驱动，落实 `runStreamJsonSession` 会话循环与 `StreamJsonController` 协作，串联输入解析、队列调度、控制请求转发与 `runNonInteractive` 调用；同时更新 `packages/cli/src/gemini.tsx` 等入口文件，让 CLI 能根据参数切换到 stream-json 会话，并在启用 `--include-partial-messages` 时输出增量事件。
   - 不包含：Hook 回调聚合、权限确认、MCP 注册执行等高级控制能力，仅聚焦“读 → 排队 → 调用 → 写”主流程与入口连通。
7. [ ] 阶段 6：Hook 回调增强  
   - 目标：在会话驱动基础上补齐 Hook 注册、触发与反馈逻辑，覆盖 `pre_tool`/`post_tool` 等事件，并在控制通道中处理中断、跳过或提示信息。
   - 不包含：权限审批与 MCP 调用，仅聚焦 Hook 生命周期与系统消息同步。
8. [ ] 阶段 7：权限与审批模式集成  
   - 目标：实现 `can_use_tool` 等权限请求流程，打通 `StreamJsonController` 与 `nonInteractiveCli` 中的审批回路，确保权限建议、允许/拒绝与提示消息均符合预期。
   - 不包含：MCP 服务器桥接与会话初始化扩展，保持任务边界聚焦在权限链路。
9. [ ] 阶段 8：MCP 桥接与资源管理  
   - 目标：按 `packages/cli/src/streamJson/session.test.ts` 规划接入 MCP 服务器注册、请求转发与回收机制，实现 `mcp_message` 等控制请求的完整数据通路。
   - 不包含：新 Hook 或权限策略调整，优先确保 MCP 生命周期与资源清理。
10. [ ] 阶段 9：文档与示例  
   - 目标：迁入 `docs/cli/index.md`、`docs/examples/stream-json`、相关 RFC 及脚本示例，依据 README 指引执行脚本与日志对比，核实协议文档与实现一致。
