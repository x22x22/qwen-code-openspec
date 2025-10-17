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

## 待确认
- `docs/rfc` 描述的 `initialize` 握手与错误语义需确认是否已有实现，若缺失需新增控制请求处理链路。
- 需要评估 `packages/core` 是否已具备事件驱动接口，或需新增转换层。
- 需要支持在未提供初始 prompt 时以 `stream-json` 模式启动并保持 CLI 常驻，直到收到显式退出（Ctrl+D/EOF 或控制请求）或发生错误。
