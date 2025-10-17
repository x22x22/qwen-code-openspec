# Spec: CLI stream-json 输出

## ADDED Requirements
### Requirement: CLI 支持 stream-json 输出格式
该需求要求 CLI MUST 在指定 stream-json 时输出符合 RFC 的 JSONL 事件。
#### Scenario: CLI 以 --output-format stream-json 运行
- `qwen code` 在解析 CLI 参数时识别 `--output-format stream-json`，并启用 JSON Lines 输出模式。
- CLI 所有向 STDOUT 写入的事件以 UTF-8 编码 JSON 对象逐行输出，每行结尾携带 `\n`。
- JSON 对象至少包含 `type` 字段，取值覆盖 `user`、`assistant`、`system`、`result`、`stream_event`、`control_request`、`control_response`、`control_cancel_request`。
- `result` 事件在对话完成或出错时发送，包含 RFC 规定的统计字段（`duration_ms`、`duration_api_ms`、`num_turns`、`session_id`、`is_error` 等）。
- `control_response` 在 CLI 回复控制通道请求时发送，`success`/`error` 字段与 RFC 对齐。
- 当 CLI 需要请求 SDK 反馈时，发送 `control_request`，并在请求被取消时发送 `control_cancel_request`。
- 以最小的编码工作量实现对应的功能，不要做任何重大重构。

### Requirement: CLI 支持 include-partial-messages 增量输出
该需求要求 CLI MUST 在启用增量消息时发送符合规范的部分事件。
#### Scenario: CLI 设置 --include-partial-messages
- `--include-partial-messages` 只能与 `--output-format stream-json` 同时使用；未选择 stream-json 时 CLI 拒绝该参数并给出错误。
- 在生成 Assistant 文本/工具内容时，CLI 发送 `stream_event` 与 `assistant` 类型的增量片段，字段遵循 RFC 的 `content_block_start`、`content_block_delta`、`message_delta` 等规范。
- CLI 在完成增量消息后发送 `stream_event` 的 `message_stop` 事件，并确保最终 `assistant` 消息包含聚合后的完整内容。
- 默认（未指定 `--include-partial-messages`）时，CLI 仅发送完整消息与必须的终止事件，不输出增量片段。

### Requirement: CLI 维持文本模式兼容性
该需求确保 CLI SHALL 在未启用 stream-json 时保留现有文本行为。
#### Scenario: CLI 未指定 stream-json 输出
- 当未传入 `--output-format stream-json` 时，CLI 保持现有文本输出行为，不引入 JSONL。
- 文本模式下忽略 `--include-partial-messages` 且提示用户使用 stream-json 模式。
