# Spec: CLI stream-json 输入

## ADDED Requirements
### Requirement: CLI 支持 stream-json 输入协议
该需求要求 CLI MUST 在启用 stream-json 输入时正确解析 JSON 行。
#### Scenario: CLI 以 --input-format stream-json 运行
- CLI 读取 STDIN 时逐行解析 UTF-8 JSON；每个对象必须包含 `type` 字段。
- `type=user` 时，CLI 将 `message` 映射为内部对话输入，支持字符串与 RFC 定义的 `ContentBlock` 数组。
- `type=user` 可携带 `parent_tool_use_id` 与一次性 `options` 字段，CLI 将其用于工具关联与临时配置。
- 当读到 EOF 时，CLI 结束当前输入会话并继续执行。
- 以最小的编码工作量实现对应的功能，不要做任何重大重构。

#### Scenario: CLI 在无初始 prompt 时常驻待输入
- 当命令行仅指定 `--input-format stream-json --output-format stream-json`（未传 `--prompt` 且 stdin 初始为空）时，CLI SHALL 保持进程运行并等待后续 JSON 行。
- 进程仅在遇到 EOF、接收到 `control_request.interrupt`/`control_cancel_request`、或发生致命错误时退出，优先与 Claude SDK 的长连接行为保持一致。
- 期间 CLI 仍需支持控制协议、工具请求等流式交互，确保可持续多轮通信。

### Requirement: CLI 处理控制通道输入
该需求要求 CLI MUST 能够响应来自 SDK 的控制请求。
#### Scenario: SDK 通过 STDIN 发送 control_request
- CLI 接收 `type=control_request` 的输入对象，并基于 `request.subtype` 执行相应逻辑（`initialize`、`interrupt`、`set_permission_mode`、`set_model` 等）。
- 对于 `initialize` 请求，CLI 在完成初始化后输出对应的 `control_response.success`，payload 包含 RFC 规定的字段。
- 对于未知或不支持的 `request.subtype`，CLI 输出 `control_response.error`，并保持进程继续运行。
- CLI 在必要时发送 `control_cancel_request` 表示放弃等待某个请求的响应。

### Requirement: CLI 报文解析错误处理
该需求确保 CLI SHALL 在输入异常时提供规范化错误反馈。
#### Scenario: 输入流包含非法 JSON 或缺失字段
- CLI 在解析 JSON 失败时输出符合 RFC 错误格式的 `result` 或 `control_response.error` 并以非零码退出。
- CLI 在检测到缺失关键字段（如 `message`、`request`）时同样返回错误，错误消息遵循 RFC 要求，帮助调用方定位问题。
