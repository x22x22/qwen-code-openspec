# Proposal: 启用 stream-json 输入输出模式

## 背景
- CLI 当前仅支持面向交互终端的文本模式，无法为 SDK 与第三方集成提供稳定的结构化协议。
- RFC `docs/rfc/qwen-code-cli-output-format-stream-json-rfc_ref_claude_clear_cn.md` 给出了与 Claude Code CLI 对齐的 JSON Lines 报文、控制通道以及增量消息规范。
- 需要在 CLI 与核心执行管线间引入 `stream-json` 输入/输出模式，并保持与现有 TUI 行为兼容。

## 目标
- 新增 `--input-format=stream-json`、`--output-format=stream-json`、`--include-partial-messages` 参数以及相关默认行为。
- 在 CLI 中实现 JSONL 报文的序列化/反序列化，与 Claude Code IPC 规范保持字段、事件类型一致。
- 扩展核心管线以识别结构化消息、控制请求/响应，并与现有执行逻辑顺畅集成。
- 覆盖必要的单元/集成测试，验证 CLI 参数解析、协议转换以及异常路径。
- 以最小的编码工作量实现对应的功能，不要做任何重大重构。

## 非目标
- 不实现新的工具/Hook/MCP 功能，只在现有能力之上暴露结构化 IO。
- 不修改 TUI 文本模式输出语义，保持向后兼容。

## 风险
- JSON 序列化错误或字段缺失可能导致 SDK 无法消费，需要严格校验与测试。
- 输入流解析错误需与 FatalError 体系对齐，避免 CLI 进程挂死。

## 里程碑
1. 规范落地：完成 OpenSpec 需求与任务拆分。
2. CLI 参数与协议实现：支持流式 JSON 输入输出。
3. 测试与文档：覆盖主要事件、错误场景并更新用户文档。
