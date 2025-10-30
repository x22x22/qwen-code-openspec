# task1-2 阶段 5-8 与 `diffs/` 对照

本文件梳理 `task1-2.md` 中尚未完成的阶段 5-8 目标，并标注与
`openspec/lightweight-tasks/diffs/` 目录下现有 `.diff` 文件之间的关联，便于后续对照与迁移。

## 阶段 5：控制器与会话驱动 + 入口切换

- `packages-cli-src-streamJson-session-ts.diff`
  - 新增 `runStreamJsonSession` 主循环：创建队列、串联输入解析、调用 `runNonInteractive` 执行以及队列回补，与阶段目标中“读 → 排队 → 调用 → 写”的主流程一致。
  - 构建 `StreamJsonController` 协作面，包括主动写入 `control_request`、响应控制事件以及中断当前运行，支撑会话驱动能力。
  - 末尾的会话清理与系统事件广播确保入口切换后的生命周期管理符合要求。
- `packages-cli-src-streamJson-controller-ts.diff`
  - 实现 `StreamJsonController` 的发送、取消、超时与中断能力，为阶段 5 中“会话循环与控制器协作”提供核心组件。
- `packages-cli-src-streamJson-types-ts.diff`
  - 新增控制信封、系统信封等结构体定义，让阶段 5 中的控制消息读写有可靠类型支撑。
- `packages-cli-src-streamJson-writer-ts.diff`
  - 提供 `StreamJsonWriter` 与增量消息构建器，负责会话循环中的助手输出、系统事件与结果信封写出，是入口切换后所有结构化输出的基础。
- `packages-cli-src-streamJson-writer-test-ts.diff`
  - 验证 writer 在增量、工具结果与系统事件场景下的行为，确保阶段 5 引入的新写出路径可测试、可回归。
- `packages-cli-src-streamJson-session-test-ts.diff`
  - 通过“先处理初始 prompt”“多 prompt 排队”“控制响应写回”等断言驱动主循环行为，直接覆盖阶段 5 的功能边界。
- `packages-cli-src-streamJson-input-ts.diff`
  - 负责读取并解析 stdin 中的 stream-json 行，同时对 `initialize` 请求即时回执，支撑阶段 5 中“串联输入解析”的要求。
- `packages-cli-src-streamJson-input-test-ts.diff`
  - 为输入解析器提供单测覆盖，包括用户消息组合、缺失消息报错与控制请求响应，保证阶段 5 集成时有足够的安全网。
- `packages-cli-src-nonInteractiveCli-ts.diff`
  - 扩展 `RunNonInteractiveOptions` 与主流程，使非交互 CLI 能在会话驱动时接入 `StreamJsonWriter`、`StreamJsonController` 与队列信号。
- `packages-cli-src-nonInteractiveCli-test-ts.diff`
  - 新增 `stream-json` 入口测试，验证在入口切换到会话模式后能按预期写出结构化信封并驱动工具流程。
- `packages-cli-src-gemini-tsx.diff`
  - 在 CLI 入口基于 `inputFormat` 分支调用 `runStreamJsonSession`，并在启用 `--include-partial-messages` 时接入增量输出，完成阶段目标中的入口切换。

> 说明：上述 diff 中亦包含 Hook、权限、MCP 相关逻辑；这些内容在阶段 6-8 中会进一步引用，此处仅标注其与阶段 5 核心循环的直接关联。

### 冲突分析结论

- 重叠文件: 阶段 5 列表中唯有 packages/cli/src/gemini.tsx 与 diffs-1aa282c 出现同名 diff，其余 streamJson/*、nonInteractiveCli* 等文件在后者目录中不存在对应改动，因此无直接冲突。
- 冲突点: diffs-1aa282c/packages-cli-src-gemini-tsx.diff 对顶部 import 进行了大规模重排，并移除了 FatalConfigError，而阶段 5 的 diff 要在旧版结构里新增 FatalInputError 与 runStreamJsonSession import；合并时该区块会产生冲突，需要在新的import 编排中手动补齐 FatalConfigError/FatalInputError 并避免重复引入 runStreamJsonSession。
- 冲突原因: 阶段 5 的 packages/cli/src/gemini.tsx 补丁基于旧版入口文件，延续了早期的 import 顺序（含FatalConfigError）并新增 FatalInputError。当前分支已经应用了 diffs-1aa282c 中的重排版本，移除了 FatalConfigError，而且顶部 import 已经引入了 runStreamJsonSession。
- 当前状态: 现有 packages/cli/src/gemini.tsx 中没有使用 FatalInputError，且 runStreamJsonSession 调用和 inputFormat=== 'stream-json' 分支已经实现（位置在 ~418 行起）。因此，阶段 5 补丁里针对 import 区块的改动在现在的代码上已无必要。
- 迁移结论: 若从 openspec/lightweight-tasks/diffs/ 迁移阶段 5 的改动到当前分支，gemini.tsx 的 import 调整无需再迁移；保持当前 import 列表即可，避免重复引入或恢复旧顺序。
- 无冲突部分: 阶段 5 中针对 main() 的 inputFormat 判断、runStreamJsonSession 调用以及 stdin 处理的增补，diffs-1aa282c未触及这些代码，可直接叠加。

## 阶段 6：Hook 回调增强

- `packages-cli-src-streamJson-session-ts.diff`
  - 新增 `registerHookCallbacks`、`handleHookCallbackRequest` 等函数，负责会话侧的 Hook 注册、匹配与控制响应，呼应阶段目标中的 Hook 生命周期管理。
- `packages-cli-src-streamJson-session-test-ts.diff`
  - 补充 `hook_callback` 场景的模拟输入与断言，验证默认决策、错误分支与系统事件输出，直接覆盖阶段 6 的 Hook 回调要求。
- `packages-cli-src-nonInteractiveCli-ts.diff`
  - 引入 `dispatchHookCallbacks`、`emitHookMessages` 等逻辑，将控制器的 `hook_callback` 请求与工具执行前后流程串联，实现阶段要求的 Hook 聚合与消息回传。
- `packages-cli-src-nonInteractiveCli-test-ts.diff`
  - 通过 Hook 相关用例（如 pre/post tool 事件）保障 CLI 侧的 Hook 调度、异常处理和系统消息输出。
- `packages-cli-src-streamJson-writer-ts.diff`
  - 通过 `emitSystemMessage` 与 `emitStreamEvent` 支撑 Hook 回调产生的系统提示与增量输出，是阶段 6 完成回写的关键依赖。
- `packages-cli-src-streamJson-controller-ts.diff`
  - 提供 `sendControlRequest`、取消与中断等能力，供 Hook 回调在 CLI 与上游宿主之间交换 `hook_callback` 事件。

### 冲突分析结论

- 直接冲突: 未发现。阶段 6 涉及的 6 个 diff（packages/cli/src/streamJson/session.ts 等）在 diffs-1aa282c 中均无同名文
  件或相同代码块改动，应用层面不存在 Git 层面的冲突行。
- 结构差异: 阶段 6 依赖的 streamJson/* 与 nonInteractiveCli.* 实现仍采用 “writer + controller + session loop” 的老架
  构，而 diffs-1aa282c 已将控制面拆分到 services/control/*、ServiceJson.ts、nonInteractiveStreamJson.ts 等新目录；若在
  该新架构上复用阶段 6 的 diff，需要改写为调用新的 ControlDispatcher/HookController 体系，而不是直接迁移原文件。

### 迁移建议

- 推荐做法: 在 diffs-1aa282c 的服务化结构中重新实现 Hook 回调（利用 ControlDispatcher + HookController），或基于现有services/control 模块补充真实逻辑；不要直接叠加阶段 6 的 streamJson/* 补丁，以免形成两套并行实现。
- 后续关注: 如需保留阶段 6 中的测试覆盖，可根据新架构在 services 层补齐等价测试而非导入原有 streamJson 测试文件。

## 阶段 7：权限与审批模式集成

- `packages-cli-src-streamJson-session-ts.diff`
  - 新增 `mapPermissionMode`、`handlePermissionModeRequest`、`handleCanUseToolRequest` 等逻辑，负责解析 `set_permission_mode` / `can_use_tool` 控制请求并与配置层交互。
- `packages-cli-src-streamJson-session-test-ts.diff`
  - 增加审批模式与权限判定的测试场景，覆盖成功更新、错误分支以及工具权限响应，确保阶段目标中的审批链路被验证。
- `packages-cli-src-nonInteractiveCli-ts.diff`
  - 实现 `handleToolPermissionRequest`、`buildPermissionSuggestions` 等函数，联动控制器与 `StreamJsonWriter` 输出权限提示，完成阶段对审批模式“建议 + 决策”的要求。
- `packages-cli-src-nonInteractiveCli-test-ts.diff`
  - 通过权限用例断言 `control_request.can_use_tool`、系统提示与审批结果写回，验证阶段 7 的集成效果。
- `packages-core-src-core-nonInteractiveToolExecutor-ts.diff`
  - 扩展 `executeToolCall` 的可选回调，暴露 `onToolCallsUpdate` 等钩子，供 CLI 在等待审批时获取实时状态并触发 `can_use_tool` 请求。
- `packages-cli-src-streamJson-controller-ts.diff`
  - 保证 CLI 能稳定发送 `set_permission_mode`、`can_use_tool` 控制请求并处理取消/超时，是审批模式闭环的传输层。

> 阶段 7 仍依赖阶段 5 中的 `StreamJsonController` 发送控制请求；相关实现详见 `packages-cli-src-streamJson-controller-ts.diff`。

### 冲突分析结论

- 触及文件: 阶段 7 涉及 streamJson/session(.test).ts、streamJson/controller.ts、nonInteractiveCli(.test).ts、core/nonInteractiveToolExecutor.ts 共 6 个 diff，全都位于 openspec/lightweight-tasks/diffs/。
- 直接冲突: diffs-1aa282c/ 中没有任何同名 diff，这些补丁不会与其产生逐行冲突。
- 架构差异: diffs-1aa282c/ 已把审批逻辑迁移到 packages/cli/src/services/control/* 与 nonInteractiveStreamJson.ts，并通过 PermissionController、ControlDispatcher 等组件实现 can_use_tool、set_permission_mode 流程；而阶段 7 的补丁仍依赖旧的 streamJson 会话循环与 StreamJsonController，两套实现语义等价但架构不同，混用会导致重复或互斥的控制流。
- 迁移建议: 若目标分支已采用 diffs-1aa282c 的服务化结构，应参照该结构重写或适配阶段 7 功能，而不是直接套用 streamJson/* 与 nonInteractiveCli 的旧式补丁；否则会出现同一能力在两套入口间分裂、审批状态不同步的问题。

## 阶段 8：MCP 桥接与资源管理

- `packages-cli-src-streamJson-session-ts.diff`
  - 新增 `handleMcpMessageRequest`、`getOrCreateMcpClient`、`closeAllMcpClients` 等函数，落实现有 MCP 服务器配置、连接复用、响应回写及资源回收，与阶段目标完全对应。
- `packages-cli-src-streamJson-session-test-ts.diff`
  - 添加 MCP 流程测试，涵盖服务器注册、请求转发、响应封装及错误处理，验证阶段 8 的通路。
- `packages-cli-src-nonInteractiveCli-ts.diff`
  - 扩展控制器回调，识别 `'mcp'` 工具调用并将结果封装成 `mcp_message` 控制请求，确保工具执行流程与会话层的 MCP 桥接保持一致。
- `packages-cli-src-nonInteractiveCli-test-ts.diff`
  - 通过 MCP 用例确认控制请求的生成、回写与错误输出，覆盖阶段 8 对资源管理和结果汇报的要求。
- `packages-cli-src-streamJson-writer-ts.diff`
  - 为 MCP 请求与响应提供统一的系统事件/结果写出能力，确保控制层在桥接外部服务器时能及时反馈会话状态。
- `packages-cli-src-streamJson-controller-ts.diff`
  - 负责向宿主发出 `mcp_message` 请求并处理取消，配合会话层管理 MCP 生命周期。
- `packages-cli-src-streamJson-types-ts.diff`
  - 定义 `mcp_message` 等控制信封结构，使得会话层与宿主之间的 MCP 事件可以类型化地序列化与反序列化。

> 结论：阶段 5-8 的核心代码与测试 diff 均集中于 `packages/cli/src/streamJson`、`packages/cli/src/nonInteractiveCli.*` 以及 CLI 入口 `gemini.tsx`，后续迁移时可按上述列表逐项比对与移植。

### 冲突分析结论

- 直接冲突: 阶段 8 涉及的 7 个 diff（streamJson/session.ts 等）在 diffs-1aa282c 中无同名补丁，合并时不会触发逐行冲突。
- 实现差异: 阶段 8 的 MCP 支撑建立在旧版 streamJson/* + StreamJsonWriter 架构上；diffs-1aa282c 已将 MCP 管理抽象到 services/control 模块（如 MCPController）并由 nonInteractiveStreamJson.ts 驱动。两套实现都提供 mcp_message/mcp_server_status 能力，但管线与类型定义不同，直接叠加会产出两套并行逻辑。
- 迁移建议: 若目标分支已采用 diffs-1aa282c 的服务化结构，应在该结构内补写阶段 8 的能力（如在 MCPController 中增强）而非直接套用 streamJson 系列 diff；否则会出现重复的 MCP 客户端缓存、控制响应和类型定义，导致维护冲突。
