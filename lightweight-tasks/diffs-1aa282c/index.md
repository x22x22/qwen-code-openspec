# Diff 索引

## .gitignore
- 更新忽略列表，新增 `QWEN.md`。
```diff
@@ -57,3 +57,4 @@ gha-creds-*.json
 
 # Log files
 patch_output.log
+QWEN.md
```

## packages/cli/package.json
- 扩充 CLI 包依赖，挂载 SDK 子包与构建脚本。
```diff
@@ -8,9 +8,20 @@
   },
   "type": "module",
   "main": "dist/index.js",
+  "types": "dist/index.d.ts",
   "bin": {
     "qwen": "dist/index.js"
   },
```

## packages/cli/src/config/config.ts
- 更新 CLI 参数解析，加入 stream-json 输入输出配置。
```diff
@@ -23,6 +23,7 @@ import {
   WriteFileTool,
   resolveTelemetrySettings,
   FatalConfigError,
+  InputFormat,
   OutputFormat,
 } from '@qwen-code/qwen-code-core';
 import type { Settings } from './settings.js';
```

## packages/cli/src/gemini.tsx
- 调整 CLI 入口，接入 stream-json 会话调度。
```diff
@@ -4,59 +4,59 @@
  * SPDX-License-Identifier: Apache-2.0
  */
 
-import React from 'react';
-import { render } from 'ink';
-import { AppContainer } from './ui/AppContainer.js';
-import { loadCliConfig, parseArguments } from './config/config.js';
```

## packages/cli/src/nonInteractiveStreamJson.ts
- 新增 stream-json 非交互主流程，实现控制消息队列。
```diff
@@ -0,0 +1,732 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/cli/src/services/MessageRouter.ts
- 新增消息路由器，负责工具与会话事件分发。
```diff
@@ -0,0 +1,111 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/cli/src/services/StreamJson.ts
- 新增 stream-json 服务层，串联读写与控制逻辑。
```diff
@@ -0,0 +1,633 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/* eslint-disable @typescript-eslint/no-explicit-any */
```

## packages/cli/src/services/control/ControlContext.ts
- 新增控制上下文，统一控制指令数据访问。
```diff
@@ -0,0 +1,73 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/cli/src/services/control/ControlDispatcher.ts
- 新增控制调度器，协调控制请求与响应。
```diff
@@ -0,0 +1,351 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/cli/src/services/control/controllers/baseController.ts
- 新增控制器基类，封装 request 处理骨架。
```diff
@@ -0,0 +1,180 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/cli/src/services/control/controllers/hookController.ts
- 新增 Hook 控制器，处理 hook 相关控制请求。
```diff
@@ -0,0 +1,56 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/cli/src/services/control/controllers/mcpController.ts
- 新增 MCP 控制器，负责工具注册与桥接。
```diff
@@ -0,0 +1,287 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/cli/src/services/control/controllers/permissionController.ts
- 新增权限控制器，管理工具执行授权流程。
```diff
@@ -0,0 +1,480 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/cli/src/services/control/controllers/systemController.ts
- 新增系统控制器，回传系统级事件与诊断。
```diff
@@ -0,0 +1,292 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/cli/src/types/protocol.ts
- 新增 CLI 协议类型声明，覆盖控制与消息结构。
```diff
@@ -0,0 +1,485 @@
+/* eslint-disable @typescript-eslint/no-explicit-any */
+
+/**
+ * Usage information types
+ */
+export interface Usage {
+  input_tokens: number;
```

## packages/core/src/config/config.ts
- 更新核心 Config，保存 stream-json 输入输出配置。
```diff
@@ -62,7 +62,7 @@ import { WriteFileTool } from '../tools/write-file.js';
 
 // Other modules
 import { ideContextStore } from '../ide/ideContext.js';
-import { OutputFormat } from '../output/types.js';
+import { InputFormat, OutputFormat } from '../output/types.js';
 import { PromptRegistry } from '../prompts/prompt-registry.js';
 import { SubagentManager } from '../subagents/subagent-manager.js';
```

## packages/core/src/output/types.ts
- 扩展输出类型定义，支持增量消息回调。
```diff
@@ -6,9 +6,15 @@
 
 import type { SessionMetrics } from '../telemetry/uiTelemetry.js';
 
+export enum InputFormat {
+  TEXT = 'text',
+  STREAM_JSON = 'stream-json',
+}
```

## packages/sdk/typescript/package.json
- 新增 SDK TypeScript 包定义与构建脚本。
```diff
@@ -0,0 +1,69 @@
+{
+  "name": "@qwen-code/sdk-typescript",
+  "version": "0.1.0",
+  "description": "TypeScript SDK for programmatic access to qwen-code CLI",
+  "main": "dist/index.js",
+  "types": "dist/index.d.ts",
+  "type": "module",
```

## packages/sdk/typescript/src/index.ts
- 暴露 SDK 入口，聚合查询与 MCP 接口。
```diff
@@ -0,0 +1,108 @@
+/**
+ * TypeScript SDK for programmatic access to qwen-code CLI
+ *
+ * @example
+ * ```typescript
+ * import { query } from '@qwen-code/sdk-typescript';
+ *
```

## packages/sdk/typescript/src/mcp/SdkControlServerTransport.ts
- 新增 SDK MCP 传输层，实现控制请求交互。
```diff
@@ -0,0 +1,153 @@
+/**
+ * SdkControlServerTransport - bridges MCP Server with Query's control plane
+ *
+ * Implements @modelcontextprotocol/sdk Transport interface to enable
+ * SDK-embedded MCP servers. Messages flow bidirectionally:
+ *
+ * MCP Server → send() → Query → control_request (mcp_message) → CLI
```

## packages/sdk/typescript/src/mcp/createSdkMcpServer.ts
- 新增 MCP Server 工厂，快速集成 SDK 控制服务。
```diff
@@ -0,0 +1,177 @@
+/**
+ * Factory function to create SDK-embedded MCP servers
+ *
+ * Creates MCP Server instances that run in the user's Node.js process
+ * and are proxied to the CLI via the control plane.
+ */
+
```

## packages/sdk/typescript/src/mcp/formatters.ts
- 新增 MCP 数据格式化工具，标准化请求响应。
```diff
@@ -0,0 +1,247 @@
+/**
+ * Tool result formatting utilities for MCP responses
+ *
+ * Converts various output types to MCP content blocks.
+ */
+
+/**
```

## packages/sdk/typescript/src/mcp/tool.ts
- 新增工具封装，简化工具定义与执行。
```diff
@@ -0,0 +1,140 @@
+/**
+ * Tool definition helper for SDK-embedded MCP servers
+ *
+ * Provides type-safe tool definitions with generic input/output types.
+ */
+
+import type { ToolDefinition } from '../types/config.js';
```

## packages/sdk/typescript/src/query/Query.ts
- 新增 Query 类，封装 SDK 查询生命周期。
```diff
@@ -0,0 +1,882 @@
+/**
+ * Query class - Main orchestrator for SDK
+ *
+ * Manages SDK workflow, routes messages, and handles lifecycle.
+ * Implements AsyncIterator protocol for message consumption.
+ */
+
```

## packages/sdk/typescript/src/query/createQuery.ts
- 新增 Query 构造函数，便捷创建查询实例。
```diff
@@ -0,0 +1,185 @@
+/**
+ * Factory function for creating Query instances.
+ */
+
+import type { CLIUserMessage } from '../types/protocol.js';
+import { serializeJsonLine } from '../utils/jsonLines.js';
+import type {
```

## packages/sdk/typescript/src/transport/ProcessTransport.ts
- 新增进程传输实现，桥接 CLI 子进程通信。
```diff
@@ -0,0 +1,496 @@
+/**
+ * ProcessTransport - Subprocess-based transport for SDK-CLI communication
+ *
+ * Manages CLI subprocess lifecycle and provides IPC via stdin/stdout using JSON Lines protocol.
+ */
+
+import { spawn, type ChildProcess } from 'node:child_process';
```

## packages/sdk/typescript/src/transport/Transport.ts
- 定义传输接口，规范跨进程通信能力。
```diff
@@ -0,0 +1,102 @@
+/**
+ * Transport interface for SDK-CLI communication
+ *
+ * The Transport abstraction enables communication between SDK and CLI via different mechanisms:
+ * - ProcessTransport: Local subprocess via stdin/stdout (initial implementation)
+ * - HttpTransport: Remote CLI via HTTP (future)
+ * - WebSocketTransport: Remote CLI via WebSocket (future)
```

## packages/sdk/typescript/src/types/config.ts
- 新增 SDK 配置类型，覆盖 CLI 路径与钩子。
```diff
@@ -0,0 +1,145 @@
+/**
+ * Configuration types for SDK
+ */
+
+import type { ToolDefinition as ToolDef } from './mcp.js';
+import type { PermissionMode } from './protocol.js';
+
```

## packages/sdk/typescript/src/types/controlRequests.ts
- 新增控制请求类型，描述控制信封结构。
```diff
@@ -0,0 +1,50 @@
+/**
+ * @license
+ * Copyright 2025 Qwen Team
+ * SPDX-License-Identifier: Apache-2.0
+ */
+
+/**
```

## packages/sdk/typescript/src/types/errors.ts
- 新增错误类型，统一 SDK 异常表达。
```diff
@@ -0,0 +1,27 @@
+/**
+ * Error types for SDK
+ */
+
+/**
+ * Error thrown when an operation is aborted via AbortSignal
+ */
```

## packages/sdk/typescript/src/types/mcp.ts
- 新增 MCP 类型定义，对齐控制协议。
```diff
@@ -0,0 +1,32 @@
+/**
+ * MCP integration types for SDK
+ */
+
+/**
+ * JSON Schema definition
+ * Used for tool input validation
```

## packages/sdk/typescript/src/types/protocol.ts
- 新增 SDK 协议类型映射，复用 CLI 协议。
```diff
@@ -0,0 +1,50 @@
+/**
+ * Protocol types for SDK-CLI communication
+ *
+ * Re-exports protocol types from CLI package to ensure SDK and CLI use identical types.
+ */
+
+export type {
```

## packages/sdk/typescript/src/utils/Stream.ts
- 新增流式工具，封装 JSON Lines 读写。
```diff
@@ -0,0 +1,157 @@
+/**
+ * Async iterable queue for streaming messages between producer and consumer.
+ */
+
+export class Stream<T> implements AsyncIterable<T> {
+  private queue: T[] = [];
+  private isDone = false;
```

## packages/sdk/typescript/src/utils/cliPath.ts
- 新增 CLI 路径工具，定位可执行文件。
```diff
@@ -0,0 +1,438 @@
+/**
+ * CLI path auto-detection and subprocess spawning utilities
+ *
+ * Supports multiple execution modes:
+ * 1. Native binary: 'qwen' (production)
+ * 2. Node.js bundle: 'node /path/to/cli.js' (production validation)
+ * 3. Bun bundle: 'bun /path/to/cli.js' (alternative runtime)
```

## packages/sdk/typescript/src/utils/jsonLines.ts
- 新增 JSON Lines 工具，解析与序列化协议。
```diff
@@ -0,0 +1,137 @@
+/**
+ * JSON Lines protocol utilities
+ *
+ * JSON Lines format: one JSON object per line, newline-delimited
+ * Example:
+ *   {"type":"user","message":{...}}
+ *   {"type":"assistant","message":{...}}
```

## packages/sdk/typescript/test/e2e/abort-and-lifecycle.test.ts
- 新增 E2E 测试，覆盖中断与生命周期流程。
```diff
@@ -0,0 +1,486 @@
+/**
+ * E2E tests based on abort-and-lifecycle.ts example
+ * Tests AbortController integration and process lifecycle management
+ */
+
+/* eslint-disable @typescript-eslint/no-unused-vars */
+
```

## packages/sdk/typescript/test/e2e/basic-usage.test.ts
- 新增 E2E 测试，验证基础调用场景。
```diff
@@ -0,0 +1,521 @@
+/**
+ * E2E tests based on basic-usage.ts example
+ * Tests message type recognition and basic query patterns
+ */
+
+import { describe, it, expect } from 'vitest';
+import { query } from '../../src/index.js';
```

## packages/sdk/typescript/test/e2e/multi-turn.test.ts
- 新增 E2E 测试，模拟多轮对话流程。
```diff
@@ -0,0 +1,519 @@
+/**
+ * E2E tests based on multi-turn.ts example
+ * Tests multi-turn conversation functionality with real CLI
+ */
+
+import { describe, it, expect } from 'vitest';
+import { query } from '../../src/index.js';
```

## packages/sdk/typescript/test/e2e/simple-query.test.ts
- 新增 E2E 测试，校验单次查询路径。
```diff
@@ -0,0 +1,744 @@
+/**
+ * End-to-End tests for simple query execution with real CLI
+ * Tests the complete SDK workflow with actual CLI subprocess
+ */
+
+/* eslint-disable @typescript-eslint/no-unused-vars */
+
```

## packages/sdk/typescript/test/unit/ProcessTransport.test.ts
- 新增单测，验证进程传输实现。
```diff
@@ -0,0 +1,207 @@
+/**
+ * Unit tests for ProcessTransport
+ * Tests subprocess lifecycle management and IPC
+ */
+
+import { describe, expect, it } from 'vitest';
+
```

## packages/sdk/typescript/test/unit/Query.test.ts
- 新增单测，覆盖 Query 行为。
```diff
@@ -0,0 +1,284 @@
+/**
+ * Unit tests for Query class
+ * Tests message routing, lifecycle, and orchestration
+ */
+
+import { describe, expect, it } from 'vitest';
+
```

## packages/sdk/typescript/test/unit/SdkControlServerTransport.test.ts
- 新增单测，验证 MCP 传输服务。
```diff
@@ -0,0 +1,259 @@
+/**
+ * Unit tests for SdkControlServerTransport
+ *
+ * Tests MCP message proxying between MCP Server and Query's control plane.
+ */
+
+import { describe, it, expect, vi, beforeEach } from 'vitest';
```

## packages/sdk/typescript/test/unit/Stream.test.ts
- 新增单测，断言流式工具行为。
```diff
@@ -0,0 +1,247 @@
+/**
+ * Unit tests for Stream class
+ * Tests producer-consumer patterns and async iteration
+ */
+
+import { describe, it, expect, beforeEach } from 'vitest';
+import { Stream } from '../../src/utils/Stream.js';
```

## packages/sdk/typescript/test/unit/cliPath.test.ts
- 新增单测，确保 CLI 路径解析稳定。
```diff
@@ -0,0 +1,668 @@
+/**
+ * Unit tests for CLI path utilities
+ * Tests executable detection, parsing, and spawn info preparation
+ */
+
+import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
+import * as fs from 'node:fs';
```

## packages/sdk/typescript/test/unit/createSdkMcpServer.test.ts
- 新增单测，覆盖 MCP Server 创建逻辑。
```diff
@@ -0,0 +1,350 @@
+/**
+ * Unit tests for createSdkMcpServer
+ *
+ * Tests MCP server creation and tool registration.
+ */
+
+import { describe, expect, it, vi } from 'vitest';
```

## packages/sdk/typescript/tsconfig.json
- 新增 TypeScript 编译配置。
```diff
@@ -0,0 +1,41 @@
+{
+  "compilerOptions": {
+    /* Language and Environment */
+    "target": "ES2022",
+    "lib": ["ES2022"],
+    "module": "ESNext",
+    "moduleResolution": "bundler",
```

## packages/sdk/typescript/vitest.config.ts
- 新增 Vitest 配置，驱动 SDK 测试套件。
```diff
@@ -0,0 +1,36 @@
+import { defineConfig } from 'vitest/config';
+import * as path from 'path';
+
+export default defineConfig({
+  test: {
+    globals: false,
+    environment: 'node',
```

## vitest.config.ts
- 更新根级 Vitest 配置，包含 SDK 子包。
```diff
@@ -6,6 +6,7 @@ export default defineConfig({
       'packages/cli',
       'packages/core',
       'packages/vscode-ide-companion',
+      'packages/sdk/typescript',
       'integration-tests',
       'scripts',
     ],
```

