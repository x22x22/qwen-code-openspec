# Diff 索引

## docs/cli/index.md
- 新增“Structured stream-json mode”小节，介绍三项流式参数并附命令示例，方便第三方以 JSON Lines 集成 CLI。
```diff
### Structured stream-json mode

For programmatic integrations, Qwen Code supports structured JSON Lines input and output:
```

## docs/examples/stream-json/README.md
- 新增英文示例文档，涵盖前置条件、快速体验、SDK 用法与自动化校验步骤。
```diff
# Stream JSON Interaction Example

This example demonstrates how to drive the Qwen Code CLI directly via the JSON Lines protocol
```

## docs/examples/stream-json/README_cn.md
- 新增中文实操指南，补充打断用例、日志定位与脚本化验证流程。
```diff
# Stream JSON 交互示例

本示例展示如何直接以 JSON Lines 协议驱动 Qwen Code CLI
```

## docs/examples/stream-json/request.jsonl
- 提供初始化握手与首条用户消息的最小 JSONL 样例，便于快速复现。
```diff
{"type":"control_request","request_id":"req-init-1","request":{"subtype":"initialize","hooks":null}}
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"请阅读 README.md 并总结三个关键特性。"}]}}
```

## docs/examples/stream-json/request_interrupt.jsonl
- 给出多轮指令与最终 `interrupt` 的请求流，演示脚本化打断流程。
```diff
{"type":"control_request","request_id":"req-init-interrupt","request":{"subtype":"initialize","hooks":null}}
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"请输出当前工作目录，并等待后续指令。"}]}}
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"请列出当前目录中的所有文件。"}]}}
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"若无更多操作，请准备结束对话。"}]}}
{"type":"control_request","request_id":"req-interrupt-final","request":{"subtype":"interrupt"}}
```

## docs/examples/stream-json/sdk.py
- 新增轻量级伪 SDK，实现命令拆解、初始化请求、控制回调与自动中断。
```diff
DEFAULT_COMMAND = (
    "npm run qwen -- --input-format stream-json --output-format stream-json "
```

## docs/examples/stream-json/simple_stream_json_client.py
- 提供最小示例脚本，直接复用伪 SDK 异步运行并触发演示。
```diff
from sdk import StreamJsonClient, StreamJsonClientOptions


async def main() -> None:
```

## docs/examples/stream-json/validate_stream_json_cli.py
- 增加自动化校验脚本，串联 `initialize`、`can_use_tool`、`hook_callback`、`mcp_message` 等场景。
```diff
"""
Automated stream-json validation harness.

This script boots the Qwen Code CLI in stream-json mode and walks through a set
```

## docs/rfc/qwen-code-agent-framework-rfc_agentic_cn.md
- 补充 Agentic 能力清单，强调会话循环、Hook 注入、MCP 注册与 SDK 责任。
```diff
# Qwen-Code Agent 框架补充清单（Agentic 加强版）

本文聚焦对现有《Qwen-Code Agent 框架架构设计（整理版）》的补充点
```

## docs/rfc/qwen-code-agent-framework-rfc_clear_cn.md
- 发布中文“整理版”架构文档，细化组件职责、治理能力与可观测方案。
```diff
# Qwen-Code Agent 框架架构设计（整理版）

## 概览

| 字段 | 详情 |
```

## docs/rfc/qwen-code-agent-framework-rfc_clear_en.md
- 对应英文“Clean Version”架构设计，服务海外团队理解 Agent SDK 总览。
```diff
# Qwen-Code Agent Framework Architecture Design (Clean Version)

## Overview

| Field | Details |
```

## docs/rfc/qwen-code-agent-framework-rfc_cn.md
- 新增基础版中文架构设计，涵盖核心概念、组件作用与系统架构图。
```diff
# Qwen-Code Agent 框架架构设计

> **设计版本**: v1.1
```

## docs/rfc/qwen-code-cli-output-format-stream-json-rfc_clear_cn.md
- 新增结构化 IO 中文“整理版”RFC，梳理协议目标、事件语义与落地计划。
```diff
# RFC: Qwen-Code CLI 结构化输入输出规范（整理版）

## 概览

| 字段 | 详情 |
```

## docs/rfc/qwen-code-cli-output-format-stream-json-rfc_clear_en.md
- 对应英文 Clean 版 RFC，面向国际合作者阐述 stream-json 协议。
```diff
# RFC: Qwen-Code CLI Structured Input/Output Specification (Clean Version)

## Overview

| Field | Details |
```

## docs/rfc/qwen-code-cli-output-format-stream-json-rfc_cn.md
- 新增中文 RFC 正文，详述 CLI 结构化协议背景、场景与实施路径。
```diff
# RFC: Qwen-Code CLI 输出格式与 IPC Stream JSON 能力

- **状态**: Draft
```

## docs/rfc/qwen-code-cli-output-format-stream-json-rfc_ref_claude_clear_cn.md
- 增补对齐 Claude 的参照版 RFC，便于对比协议字段与行为差异。
```diff
# RFC: Qwen-Code CLI 结构化输入输出规范（整理版）

## 概览

| 字段 | 详情 |
```

## dot-gitignore.diff
- 更新忽略规则，新增 Python 缓存与字节码排除项以支撑示例脚本。
```diff
__pycache__/
*.py[codz]
*$py.class
```

## package.json
- 增加 `qwen` 与 `stream-json-session` 脚本，方便直接触发 CLI 及持久会话。
```diff
"qwen": "tsx packages/cli/index.ts",
"stream-json-session": "tsx packages/cli/index.ts --input-format stream-json --output-format stream-json",
```

## packages/cli/src/config/config.test.ts
- 扩充单测，校验 `--include-partial-messages` 依赖 stream-json 输出及配置透传。
```diff
expect(mockConsoleError).toHaveBeenCalledWith(
  expect.stringContaining(
    '--include-partial-messages requires --output-format stream-json',
  ),
);
```

## packages/cli/src/config/config.ts
- CLI 参数解析支持 `input/output-format` 与增量消息校验，并将配置写入核心 Config。
```diff
.option('input-format', {
  type: 'string',
  choices: ['text', 'stream-json'],
```

## packages/cli/src/gemini.tsx
- CLI 入口检测 stream-json 输入格式，交由 `runStreamJsonSession` 处理，并接受空起始提示。
```diff
const inputFormat = config.getInputFormat();
if (inputFormat === 'stream-json') {
  const trimmedInput = input.trim();
  const nonInteractiveConfig = await validateNonInteractiveAuth(
```

## packages/cli/src/nonInteractiveCli.test.ts
- 大幅扩展单测，验证 stream-json 输出、增量事件、工具结果与控制流程。
```diff
it('should emit stream-json envelopes when output format is stream-json', async () => {
  (mockConfig.getOutputFormat as vi.Mock).mockReturnValue('stream-json');
```

## packages/cli/src/nonInteractiveCli.ts
- 重构非交互模式，增加 `RunNonInteractiveOptions`、权限建议、增量输出与工具结果桥接。
```diff
export interface RunNonInteractiveOptions {
  abortController?: AbortController;
  streamJson?: {
    writer: StreamJsonWriter;
```

## packages/cli/src/streamJson/controller.ts
- 新增控制器，负责发送 `control_request`、处理响应/取消并支持主动终止运行。
```diff
sendControlRequest(
  subtype: string,
  payload: Record<string, unknown>,
```

## packages/cli/src/streamJson/input.test.ts
- 引入输入解析单测，覆盖初始化响应、用户消息合并与缺失消息报错。
```diff
describe('parseStreamJsonInputFromIterable', () => {
  it('parses user messages from stream-json lines', async () => {
```

## packages/cli/src/streamJson/input.ts
- 实现 stream-json 输入读取器，自动回应 `initialize` 并提取用户 prompt。
```diff
export async function parseStreamJsonInputFromIterable(
  lines: AsyncIterable<string>,
  emitEnvelope: (envelope: StreamJsonOutputEnvelope) => void = writeEnvelope,
```

## packages/cli/src/streamJson/session.test.ts
- 为会话主流程编写单测，验证队列处理、控制请求与 MCP 注册等逻辑。
```diff
describe('runStreamJsonSession', () => {
  beforeEach(() => {
    vi.clearAllMocks();
```

## packages/cli/src/streamJson/session.ts
- 新增核心会话循环，串联输入解析、队列调度、控制通道与 `runNonInteractive` 执行。
```diff
export async function runStreamJsonSession(
  config: Config,
  initialPrompt: string | undefined,
```

## packages/cli/src/streamJson/types.ts
- 定义 stream-json 协议类型，包含消息块、控制信封、流事件与序列化辅助。
```diff
export type StreamJsonContentBlock =
  | StreamJsonTextBlock
  | StreamJsonThinkingBlock
```

## packages/cli/src/streamJson/writer.test.ts
- 新增 writer 单测，覆盖结果统计、思维增量、工具输入与系统事件。
```diff
describe('StreamJsonWriter', () => {
  let writes: string[];

  beforeEach(() => {
```

## packages/cli/src/streamJson/writer.ts
- 实现 StreamJsonWriter，支持构建助手消息、输出流事件、工具结果与最终统计。
```diff
export class StreamJsonWriter {
  private readonly includePartialMessages: boolean;
  private readonly sessionId: string;
```

## packages/core/src/config/config.ts
- 核心配置记录输入/输出格式与增量标记，并提供访问器供 CLI 查询。
```diff
getInputFormat(): 'text' | 'stream-json' {
  return this.inputFormat;
}
```

## packages/core/src/core/nonInteractiveToolExecutor.ts
- 扩展工具执行入口，允许传入输出更新与完成回调，方便 stream-json 桥接。
```diff
export interface ExecuteToolCallOptions {
  outputUpdateHandler?: OutputUpdateHandler;
  onAllToolCallsComplete?: AllToolCallsCompleteHandler;
```

## scripts/build_package.js
- 构建脚本改用 `spawnSync` 调用 `tsc --build`，在诊断失败时仅发出警告继续流程。
```diff
const tscResult = spawnSync('tsc', ['--build'], { encoding: 'utf8' });
if (tscResult.status !== 0) {
  console.warn(
```
