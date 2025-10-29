---
title: stream-json 功能变更理解记录
status: [x]
progress:
- 2025-10-29 17:07:30: 汇总 enable-stream-json-io 变更与示例文档的完整理解，供后续断点续跑参考。
---

## 背景脉络

- 本次迁移来自 `@openspec/changes/enable-stream-json-io/`，目标是在 CLI 中新增 `stream-json` 输入/输出模式，与 Claude Code CLI 的 JSON Lines 协议对齐，满足第三方 SDK 集成需求。
- 变更强调最小化重构：在保留原文本模式兼容性的前提下，通过新增参数与事件管线实现结构化 IO。
- 开发参考文档集中于 `docs/rfc/qwen-code-cli-output-format-stream-json-*.md` 系列以及 `docs/examples/stream-json` 示例目录。

## CLI 参数与运行模式

- 新增参数：
  - `--input-format stream-json`: 切换 STDIN 解析为 JSON Lines，持续监听用户消息与控制请求。
  - `--output-format stream-json`: 将 STDOUT 切换为 JSON Lines 事件流（包含 `system`、`assistant`、`user`、`result`、`stream_event`、`control_request`、`control_response` 等）。
  - `--include-partial-messages`: 仅在 `--output-format stream-json` 启用时有效，用于输出 `message_start/content_block_delta/message_stop` 等增量事件。
- 默认情况下 CLI 仍以文本模式运行；若用户误用 `--include-partial-messages` 而未启用 stream-json 输出，需要报错提示。
- stream-json 模式允许常驻运行（无初始 prompt 时不立即退出），支持多轮输入和控制通道交互。

## 输入侧核心要点

- STDIN 逐行读取 UTF-8 JSON，对象必须带 `type` 字段；`type=user` 映射为内部对话消息，支持 `ContentBlock` 数组格式。
- 支持可选字段：
  - `parent_tool_use_id`：与工具调用链路关联。
  - `options`：一次性会话配置（如临时模型）。
- `type=control_request` 用于能力协商与执行控制，支持的 `subtype` 包括 `initialize`、`interrupt`、`can_use_tool`、`hook_callback`、`mcp_message`、`set_permission_mode`、`set_model` 等；未知 subtype 需返回 `control_response.error` 但进程保持运行。
- 输入解析错误（非法 JSON/缺失字段）需要以规范化 `result` 或 `control_response.error` 反馈，并以非零码退出。

## 输出侧核心要点

- 所有事件以 JSON Lines 输出，关键类型及内容：
  - `system`: 初始化或状态变化时输出环境、模型、可用工具、slash 命令、权限模式等元数据，与 RFC 字段对齐。
  - `assistant`: 聚合后的完整回复内容，包括 `text`、`tool_use` 等。
  - `stream_event`: 在启用 `--include-partial-messages` 时输出增量片段（`message_start`、`content_block_start/delta/stop`、`message_stop`，涵盖 `text_delta`、`thinking_delta`、`input_json_delta` 等类型）。
  - `result`: 会话结束或报错时输出统计（`duration_ms`、`duration_api_ms`、`num_turns`、`usage`、`total_cost_usd`、`session_id`、`is_error` 等）。
  - `control_response`: 回复控制请求，包含 `success`、`response` 或 `error`。
  - `control_request`/`control_cancel_request`: CLI 主动向上游请求反馈（如工具许可），或取消已发出的控制请求。
- 错误场景需输出 `is_error=true` 的 `result`，并携带格式化错误信息。

## 代码结构理解

- **新目录 `packages/cli/src/streamJson/`**：
  - `types.ts`: 定义 stream-json 协议相关类型（Envelope、ContentBlock、Usage、控制上下文等）。
  - `writer.ts`: 封装所有事件写出逻辑，负责系统事件、增量片段、结果汇总等。
  - `session.ts`: 管理长链接会话、控制请求处理、Hook 回调、工具调用确认等，并维护 `StreamJsonControlContext`。
  - `controller.ts`: 提供高层控制器接口，协调 CLI 主循环与 stream-json 会话状态（包括打断、请求取消等）。
  - `input.ts`: 解析 STDIN JSON 行并转换为内部事件；配套 `input.test.ts` 覆盖合法/非法输入与控制请求处理。
- **`packages/cli/src/nonInteractiveCli.ts` 主要变化**：
  - 新增 `RunNonInteractiveOptions`，注入 `streamJson` 写出器、控制器、工具调用登记表、控制上下文。
  - 扩展工具调用执行流程：支持确认 Hook（含 `ToolConfirmationOutcome`），记录权限建议（exec/diff/plan/mcp/info），在 stream-json 模式下输出 `tool_result` 事件。
  - 统一错误处理：使用 `parseAndFormatApiError` 生成结构化报错，stream-json 模式下同步写入失败的 `result`。
  - 记录统计信息：对接 Gemeni 客户端，用于计算 `usage`、API 时长与成本估算。
- **配置与核心改动**：
  - `packages/cli/src/config/config.ts` & `config.test.ts`: 增加 CLI 参数解析，校验 stream-json 模式与部分消息选项。
  - `packages/cli/src/nonInteractiveCli.test.ts`: 大幅扩充测试用例，覆盖 stream-json 输出、工具回执、控制通道交互、错误场景等。
  - `packages/core/src/config/config.ts`：新增 stream-json 相关默认项（如 includePartialMessages、输出模式等）。
  - `packages/core/src/core/nonInteractiveToolExecutor.ts`：对齐工具执行数据结构，支持回传给 stream-json 管线。

## 示例与测试理解

- `docs/examples/stream-json/README_cn.md`：
  - 给出基础体验命令、常驻运行示例、打断流程、控制请求（`can_use_tool`/`mcp_message`/`tool_result`）演示。
  - 列举脚本化验证方式（`simple_stream_json_client.py`、`validate_stream_json_cli.py`）及期待日志输出。
- 示例文件：
  - `request.jsonl` 与 `request_interrupt.jsonl`：示例输入流。
  - `sdk.py`：伪 SDK，封装 `StreamJsonClient`，自动处理控制请求与打断逻辑，可作为对接参考。
  - `simple_stream_json_client.py`：直接运行 SDK 客户端 Demo。
  - `validate_stream_json_cli.py`：脚本化验证多种控制子类型和工具链路，可通过环境变量切换场景。
  - `logs/1.log` 等：提供 CLI 实际输出样例。

## 其他文档补充

- `docs/rfc/qwen-code-cli-output-format-stream-json-*.md`：详细描述事件字段、控制通道、增量消息生命周期，与 Claude 协议对齐。
- 相关 Agent Framework RFC (`docs/rfc/qwen-code-agent-framework-rfc_*`) 对整体架构、Hook/工具/MCP 交互提供上下文，说明 stream-json 如何嵌入整个系统事件流。

## 风险与待关注点

- 旧分支存在大量文档与版本号回退，迁移时需筛选，避免影响主线版本或引入过时配置。
- 控制通道子类型覆盖面广（尤其 `mcp_message`、hook 相关）——迁入时需确认核心层已有支撑逻辑，否则至少维持与旧分支一致的降级行为。
- 增量事件顺序与字段需严格对齐 RFC，测试时要比对 `docs/examples/stream-json/logs/` 中的实际输出。

## 验证方法回顾

- 单元测试：`npm run test -- packages/cli/src/...` 可验证输入解析、会话控制、Writer 输出等；README 中点名的测试需全部通过。
- 实例脚本：运行 `python docs/examples/stream-json/simple_stream_json_client.py` 或 `validate_stream_json_cli.py`，对照日志判断协议实现是否一致。
- 手工验证：按照 README 提供的命令行步骤，对 `initialize`、增量事件、工具调用、MCP 消息、`tool_result` 回链等场景逐项检查。
