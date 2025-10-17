# Design: CLI stream-json 输入输出

## 架构概览
- CLI 层：在 `packages/cli` 中扩展命令行参数解析，根据 `--input-format`、`--output-format`、`--include-partial-messages` 配置切换运行模式。
- 核心层：`packages/core` 负责执行管线，需新增流式 JSON 事件生产/消费接口，封装消息与控制通道，确保与 TUI 共享上下文。
- 通信协议：严格遵循 `docs/rfc/qwen-code-cli-output-format-stream-json-rfc_ref_claude_clear_cn.md` 与 `docs/rfc/claude_code_ipc.md`，复用字段命名与事件语义。

## 关键决策
1. **协议映射**：采用统一的事件枚举与 payload 模型，将核心内部事件序列化为 JSONL；对输入流则解析为内部命令对象。
2. **兼容性策略**：默认保持文本模式，只有当显式指定 `stream-json` 时才启用结构化通道；未指定输入格式时保留原 stdin 行读取行为。
3. **部分消息**：当启用 `--include-partial-messages` 时，核心在生成 `assistant` 动态内容时发送 `stream_event` 与部分 `assistant` 内容；否则仅发送完整消息。
4. **错误处理**：解析失败时抛出带 exit code 的 `FatalError`，输出符合规范的 `control_response.error` 或 `result.is_error=true` 事件，以便 SDK 感知。

## 实施进展
- CLI 现已在 `--input-format/--output-format stream-json` 下走 JSONL 管线，支持 `--include-partial-messages` 的增量事件。
- 新增 `StreamJsonWriter`、`runStreamJsonSession`，可在无初始 prompt 时常驻等待多轮输入，处理 `control_request.initialize` 与 `control_request.interrupt`。
- 非交互模式链路复用现有 `runNonInteractive`，输出 `assistant`、`result` 事件并回写工具结果/错误。
- 补充单元测试覆盖参数解析、输入解析、常驻会话与控制请求。
- 已交付的测试脚本包括：
  - `packages/cli/src/config/config.test.ts`（参数解析、模式判定）
  - `packages/cli/src/nonInteractiveCli.test.ts`（stream-json 输出、增量与工具回执）
  - `packages/cli/src/streamJson/input.test.ts`（输入解析与错误处理）
  - `packages/cli/src/streamJson/session.test.ts`（常驻会话、多轮消息、中断控制）

## 待确认
- `docs/rfc` 描述的完整控制通道仍待补齐：CLI 侧尚未实现 `can_use_tool`、`hook_callback`、`mcp_message` 等 `control_request` 子类型的桥接。
- 需要评估 `packages/core` 是否已具备事件驱动接口，或需新增转换层，以便输出 `system` 初始化消息、`usage/duration` 统计等高级字段。
- `stream_event` 当前仅覆盖文本增量，后续若要对齐 Claude 还需支持 `thinking_delta`、工具调用增量、`message_start`/`content_block_*` 的完整生命周期封装。
