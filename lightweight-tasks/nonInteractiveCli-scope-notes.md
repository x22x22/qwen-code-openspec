› 请按pr-comment.md的建议执行，并且把剔除的内容在 @openspec/lightweight-tasks/task1-2-4.md 中进行备忘。


• I’m aligning the code with the PR comment to minimize changes in nonInteractive CLI by focusing on core stream-json
  and control request/response features only. This means removing hooking, permission handling, and advanced logic,
  while keeping minimal streaming functionality and adjusting related tests and documentation accordingly.

• Explored
  └ Read nonInteractiveCli.ts

• I’m planning a major cleanup to remove several unused types and functions like ExecuteToolCallOptions and
  convertToFunctionResponse, simplifying stream-json handling by keeping only needed parts like StreamJsonController for
  aborts. I’ll reorganize imports, streamline runNonInteractive by removing permission hooking and complex controls, and
  create simpler helpers to convert user envelopes to Part arrays. Alongside, I’ll adjust related session code and tests
  to fit the new, cleaner structure while ensuring proper option and settings passing throughout. This overhaul aims to
  reduce complexity and improve clarity in the streaming and tool call logic.

• Explored
  └ Read nonInteractiveCli.test.ts

• Edited packages/cli/src/nonInteractiveCli.test.ts (+0 -23)
    11    SessionMetrics,
    12 -  ToolCallRequestInfo,
    12  } from '@qwen-code/qwen-code-core';
       ⋮
    20    FatalInputError,
    22 -  ToolConfirmationOutcome,
    23 -  convertToFunctionResponse,
    21  } from '@qwen-code/qwen-code-core';
       ⋮
    25  import type { LoadedSettings } from './config/settings.js';
    29 -import type { StreamJsonUserEnvelope } from './streamJson/types.js';
    30 -import type { StreamJsonController } from './streamJson/controller.js';
    26
       ⋮
    56 -      .mockImplementation(
    57 -        (_toolName: string, callId: string, content: unknown) => [
    58 -          { text: `converted-${callId}-${JSON.stringify(content)}` },
    59 -        ],
    60 -      ),
    49    };
       ⋮
    58  }));
    71 -
    72 -type AwaitingApprovalToolCall = {
    73 -  status: 'awaiting_approval';
    80 -  } & Record<string, unknown>;
    81 -};
    59

• Explored
    77      mockShutdownTelemetry = vi.mocked(shutdownTelemetry);
    78 -    vi.mocked(convertToFunctionResponse).mockClear();
    79 -    vi.mocked(convertToFunctionResponse).mockImplementation(
    80 -      (_toolName: string, callId: string, content: unknown) => [
    81 -        { text: `converted-${callId}-${JSON.stringify(content)}` },
    82 -      ],
    83 -    );
    84 -
    78      mockCommandServiceCreate.mockResolvedValue({

• Explored
  └ Search remote in nonInteractiveCli.test.ts
    Search temporary model in nonInteractiveCli.test.ts
    Search toHaveBeenCalledWith\( in nonInteractiveCli.test.ts
    Read nonInteractiveCli.test.ts

• Edited packages/cli/src/nonInteractiveCli.test.ts (+0 -1)
    246        expect.any(AbortSignal),
    247 -      undefined,
    247      );

• Explored
  └ Read nonInteractiveCli.test.ts

• Edited packages/cli/src/nonInteractiveCli.test.ts (+0 -1)
    587        expect.any(AbortSignal),
    588 -      undefined,
    588      );

• Edited packages/cli/src/nonInteractiveCli.test.ts (+0 -117)
     945
     946 -  it('should convert remote tool results from stream-json user envelopes', async () => {
     947 -    (mockConfig.getOutputFormat as vi.Mock).mockReturnValue('stream-json');
     948 -    (mockConfig.getIncludePartialMessages as vi.Mock).mockReturnValue(false);
     949 -
     950 -    const toolCallRegistry = new Map<string, ToolCallRequestInfo>();
     951 -    toolCallRegistry.set('tool-remote', {
     952 -      callId: 'tool-remote',
     953 -      name: 'remote_tool',
     954 -      args: { input: 'payload' },
     955 -      isClientInitiated: false,
     956 -      prompt_id: 'prompt-remote',
     957 -    } as unknown as ToolCallRequestInfo);
     958 -
     959 -    const mockController = {
     960 -      sendControlRequest: vi.fn(),
     961 -      setActiveRunAbortController: vi.fn(),
     962 -      handleControlResponse: vi.fn(),
     963 -      handleControlCancel: vi.fn(),
     964 -      interruptActiveRun: vi.fn(),
     965 -    } as unknown as StreamJsonController;
     966 -
     967 -    const writes: string[] = [];
     968 -    processStdoutSpy.mockImplementation((chunk: string | Uint8Array) => {
     969 -      if (typeof chunk === 'string') {
     970 -        writes.push(chunk);
     971 -      } else {
     972 -        writes.push(Buffer.from(chunk).toString('utf8'));
     973 -      }
     974 -      return true;
     975 -    });
     976 -
     977 -    mockGeminiClient.sendMessageStream.mockReturnValue(
     978 -      createStreamFromEvents([
     979 -        { type: GeminiEventType.Content, value: 'Processed' },
     980 -      ]),
     981 -    );
     982 -
     983 -    await runNonInteractive(
     984 -      mockConfig,
     985 -      mockSettings,
     986 -      'ignored input',
     987 -      'prompt-remote',
     988 -      {
     989 -        streamJson: {
     990 -          controller: mockController,
     991 -          toolCallRegistry,
     992 -        },
     993 -        userEnvelope: {
     994 -          type: 'user',
     995 -          message: {
     996 -            content: [
     997 -              {
     998 -                type: 'tool_result',
     999 -                tool_use_id: 'tool-remote',
    1000 -                content: 'processed output',
    1001 -              },
    1002 -            ],
    1003 -          },
    1004 -        } as unknown as StreamJsonUserEnvelope,
    1005 -      },
    1006 -    );
    1007 -
    1008 -    expect(vi.mocked(convertToFunctionResponse)).toHaveBeenCalledWith(
    1009 -      'remote_tool',
    1010 -      'tool-remote',
    1011 -      'processed output',
    1012 -    );
    1013 -    expect(toolCallRegistry.size).toBe(0);
    1014 -    expect(mockGeminiClient.sendMessageStream).toHaveBeenCalledWith(
    1015 -      [{ text: 'converted-tool-remote-"processed output"' }],
    1016 -      expect.any(AbortSignal),
    1017 -      'prompt-remote',
    1018 -    );
    1019 -
    1020 -    const envelopes = writes
    1021 -      .join('')
    1022 -      .split('\n')
    1023 -      .filter((line) => line.trim().length > 0)
    1024 -      .map((line) => JSON.parse(line));
    1025 -    expect(envelopes.at(-1)?.type).toBe('result');
    1026 -  });
    1027 -
    1028 -  it('applies temporary model overrides from stream-json envelope options', async () => {
    1029 -    (mockConfig.getOutputFormat as vi.Mock).mockReturnValue('stream-json');
    1030 -
    1031 -    mockGeminiClient.sendMessageStream.mockReturnValue(
    1032 -      createStreamFromEvents([
    1033 -        { type: GeminiEventType.Content, value: 'Response' },
    1034 -      ]),
    1035 -    );
    1036 -
    1037 -    await runNonInteractive(
    1038 -      mockConfig,
    1039 -      mockSettings,
    1040 -      'ignored',
    1045 -          message: { content: 'Hello via envelope' },
    1046 -          options: { temporary_model: 'temp-model' },
    1047 -        } as unknown as StreamJsonUserEnvelope,
    1048 -      },
    1049 -    );
    1050 -
    1051 -    expect(mockConfig.setModel).toHaveBeenNthCalledWith(
    1052 -      1,
    1053 -      'temp-model',
    1054 -      expect.objectContaining({ context: 'temporary_model' }),
    1055 -    );
    1056 -    expect(mockConfig.setModel).toHaveBeenNthCalledWith(
    1057 -      2,
    1058 -      'test-model',
    1059 -      expect.objectContaining({ context: 'temporary_model_restore' }),
    1060 -    );
    1061 -  });
    1062 -
     946    it('should include usage metadata and API duration in stream-json result', async () => {

• Explored
  └ Search convertToFunctionResponse in nonInteractiveCli.test.ts
    Search StreamJsonUserEnvelope in nonInteractiveCli.test.ts

• Edited packages/cli/src/nonInteractiveCli.ts (+15 -119)
     76
     77 -function toolResultContentToPartList(
     78 -  content?: StreamJsonContentBlock[] | string,
     79 -): PartListUnion {
     80 -  if (!content) {
     81 -    return [{ text: '' }];
     77 +function extractPartsFromEnvelope(
     78 +  envelope: StreamJsonUserEnvelope | undefined,
     79 +): PartListUnion | null {
     80 +  if (!envelope) {
     81 +    return null;
     82    }
     83
     84 +  const content = envelope.message?.content;
     85    if (typeof content === 'string') {
     86      return content;
     86 -  }
     87 -
     88 -  const parts: Part[] = [];
     89 -  for (const item of content) {
     90 -    if (!item || typeof item !== 'object' || !('type' in item)) {
     91 -      continue;
     92 -    }
     93 -    if (item.type === 'text') {
     94 -      parts.push({ text: item.text ?? '' });
     95 -    } else {
     96 -      parts.push({ text: JSON.stringify(item) });
     97 -    }
     87    }
     99 -
    100 -  return parts.length > 0 ? parts : [{ text: '' }];
    101 -}
    102 -
    103 -function buildToolResultFallbackText(
    104 -  block: Extract<StreamJsonContentBlock, { type: 'tool_result' }>,
    105 -): string | undefined {
    106 -  if (typeof block.content === 'string') {
    107 -    return block.content;
    108 -  }
    109 -
    110 -  if (Array.isArray(block.content)) {
    111 -    const fragments: string[] = [];
    112 -    for (const item of block.content) {
    113 -      if (!item || typeof item !== 'object' || !('type' in item)) {
     88 +  if (Array.isArray(content)) {
     89 +    const parts: Part[] = [];
     90 +    for (const block of content) {
     91 +      if (!block || typeof block !== 'object' || !('type' in block)) {
     92          continue;
     93        }
    116 -      if (item.type === 'text') {
    117 -        if (item.text) {
    118 -          fragments.push(item.text);
    119 -        }
     94 +      if (block.type === 'text' && block.text) {
     95 +        parts.push({ text: block.text });
     96        } else {
    121 -        fragments.push(JSON.stringify(item));
     97 +        parts.push({ text: JSON.stringify(block) });
     98        }
     99      }
    124 -    if (fragments.length > 0) {
    125 -      return fragments.join('');
    126 -    }
    100 +    return parts.length > 0 ? parts : null;
    101    }
    128 -
    129 -  if (block.is_error) {
    130 -    return 'Tool execution reported an error without content.';
    131 -  }
    132 -
    133 -  return undefined;
    134 -}
    135 -
    136 -function buildPermissionSuggestions(confirmationDetails: unknown): unknown {
    137 -  if (
    138 -    !confirmationDetails ||
    139 -    typeof confirmationDetails !== 'object' ||
    140 -    !('type' in confirmationDetails)
    141 -  ) {
    142 -    return null;
    143 -  }
    144 -
    145 -  const details = confirmationDetails as Record<string, unknown>;
    146 -  const type = String(details['type'] ?? '');
    147 -  const title =
    148 -    typeof details['title'] === 'string' ? details['title'] : undefined;
    149 -
    150 -  switch (type) {
    151 -    case 'exec':
    152 -      return [
    153 -        {
    154 -          type: 'command',
    155 -          title,
    156 -          command: details['command'],
    157 -          root_command: details['rootCommand'],
    158 -        },
    159 -      ];
    160 -    case 'edit':
    161 -      return [
    162 -        {
    163 -          type: 'diff',
    164 -          title,
    165 -          file_name: details['fileName'],
    166 -          file_path: details['filePath'],
    167 -          diff: details['fileDiff'],
    168 -          original_content: details['originalContent'],
    169 -          new_content: details['newContent'],
    170 -        },
    171 -      ];
    172 -    case 'plan':
    173 -      return [
    174 -        {
    175 -          type: 'plan',
    176 -          title,
    177 -          plan: details['plan'],
    178 -        },
    179 -      ];
    180 -    case 'mcp':
    181 -      return [
    182 -        {
    183 -          type: 'mcp',
    184 -          title,
    185 -          server_name: details['serverName'],
    190 -    case 'info':
    191 -      return [
    192 -        {
    193 -          type: 'info',
    194 -          title,
    195 -          prompt: details['prompt'],
    196 -          urls: details['urls'],
    197 -        },
    198 -      ];
    199 -    default:
    200 -      return [
    201 -        {
    202 -          type,
    203 -          title,
    204 -        },
    205 -      ];
    206 -  }
    102 +  return null;
    103  }

• Explored
  └ Search applyTemporaryEnvelopeOptions in nonInteractiveCli.ts
    Read nonInteractiveCli.ts

• Edited packages/cli/src/nonInteractiveCli.ts (+0 -44)
    168
    169 -async function applyTemporaryEnvelopeOptions(
    170 -  config: Config,
    171 -  envelope?: StreamJsonUserEnvelope,
    172 -): Promise<() => Promise<void>> {
    173 -  const restoreStack: Array<() => Promise<void>> = [];
    174 -  const rawOptions = envelope?.options;
    175 -
    176 -  if (rawOptions && typeof rawOptions === 'object') {
    177 -    const options = rawOptions as Record<string, unknown>;
    178 -    const temporaryModelRaw = options['temporary_model'];
    179 -    if (typeof temporaryModelRaw === 'string') {
    180 -      const temporaryModel = temporaryModelRaw.trim();
    181 -      if (temporaryModel.length > 0) {
    182 -        const originalModel = config.getModel();
    183 -        if (temporaryModel !== originalModel) {
    184 -          await config.setModel(temporaryModel, {
    185 -            reason: 'stream-json-options',
    186 -            context: 'temporary_model',
    187 -          });
    188 -          restoreStack.push(async () => {
    189 -            try {
    190 -              await config.setModel(originalModel, {
    195 -              console.debug(
    196 -                'Failed to restore model after temporary stream-json option:',
    197 -                error,
    198 -              );
    199 -            }
    200 -          });
    201 -        }
    202 -      }
    203 -    }
    204 -  }
    205 -
    206 -  return async () => {
    207 -    for (const restore of restoreStack.reverse()) {
    208 -      await restore();
    209 -    }
    210 -  };
    211 -}
    212 -
    169  function extractUsageFromGeminiClient(

• Explored
  └ Search cleanupTemporaryOptions in nonInteractiveCli.ts
    Read nonInteractiveCli.ts

• I’m weighing rewriting the entire file from scratch to simplify and ensure only needed parts remain, keeping key
  functions, imports, and logic for non-interactive CLI behavior including JSON and stream JSON modes. Using git to
  revert or patch seems tricky, so deleting and replacing with a carefully rebuilt minimal version feels safer to
  avoid leftover or broken code. I’ll base it on the earlier simplified function and include necessary helpers, error
  handling, and commands to maintain expected functionality.

• Deleted packages/cli/src/nonInteractiveCli.ts (+0 -859)
      1 -/**
      2 - * @license
      3 - * Copyright 2025 Google LLC
      4 - * SPDX-License-Identifier: Apache-2.0
      5 - */
      6 -
      7 -import type {
      8 -  Config,
      9 -  ToolCallRequestInfo,
     10 -  ExecuteToolCallOptions,
     11 -} from '@qwen-code/qwen-code-core';
     12 -import { isSlashCommand } from './ui/utils/commandUtils.js';
     13 -import type { LoadedSettings } from './config/settings.js';
     14 -import {
     15 -  executeToolCall,
     16 -  shutdownTelemetry,
     17 -  isTelemetrySdkInitialized,
     18 -  GeminiEventType,
     19 -  FatalInputError,
     20 -  promptIdContext,
     21 -  OutputFormat,
     22 -  JsonFormatter,
     23 -  uiTelemetryService,
     24 -  convertToFunctionResponse,
     25 -  ToolConfirmationOutcome,
     26 -} from '@qwen-code/qwen-code-core';
     27 -
     28 -import type { Content, Part, PartListUnion } from '@google/genai';
     29 -import { StreamJsonWriter } from './streamJson/writer.js';
     30 -import type {
     31 -  StreamJsonContentBlock,
     32 -  StreamJsonUsage,
     33 -  StreamJsonUserEnvelope,
     34 -} from './streamJson/types.js';
     35 -import type { StreamJsonController } from './streamJson/controller.js';
     36 -import type { StreamJsonControlContext } from './streamJson/session.js';
     37 -
     38 -import { handleSlashCommand } from './nonInteractiveCliCommands.js';
     39 -import { ConsolePatcher } from './ui/utils/ConsolePatcher.js';
     40 -import { handleAtCommand } from './ui/hooks/atCommandProcessor.js';
     41 -import {
     42 -  handleError,
     43 -  handleToolError,
     44 -  handleCancellationError,
     45 -  handleMaxTurnsExceededError,
     46 -} from './utils/errors.js';
     47 -
     48 -export interface RunNonInteractiveOptions {
     49 -  abortController?: AbortController;
     50 -  streamJson?: {
     51 -    writer?: StreamJsonWriter;
     52 -    controller?: StreamJsonController;
     53 -    toolCallRegistry?: Map<string, ToolCallRequestInfo>;
     54 -    controlContext?: StreamJsonControlContext;
     55 -  };
     56 -  userEnvelope?: StreamJsonUserEnvelope;
     57 -}
     58 -
     59 -function normalizePartList(parts: PartListUnion | null): Part[] {
     60 -  if (!parts) {
     61 -    return [];
     62 -  }
     63 -
     64 -  if (typeof parts === 'string') {
     65 -    return [{ text: parts }];
     66 -  }
     67 -
     68 -  if (Array.isArray(parts)) {
     69 -    return parts.map((part) =>
     70 -      typeof part === 'string' ? { text: part } : (part as Part),
     71 -    );
     72 -  }
     73 -
     74 -  return [parts as Part];
     75 -}
     76 -
     77 -function extractPartsFromEnvelope(
     78 -  envelope: StreamJsonUserEnvelope | undefined,
     79 -): PartListUnion | null {
     80 -  if (!envelope) {
     81 -    return null;
     82 -  }
     83 -
     84 -  const content = envelope.message?.content;
     85 -  if (typeof content === 'string') {
     86 -    return content;
     87 -  }
     88 -  if (Array.isArray(content)) {
     89 -    const parts: Part[] = [];
     90 -    for (const block of content) {
     91 -      if (!block || typeof block !== 'object' || !('type' in block)) {
     92 -        continue;
     93 -      }
     94 -      if (block.type === 'text' && block.text) {
     95 -        parts.push({ text: block.text });
     96 -      } else {
     97 -        parts.push({ text: JSON.stringify(block) });
     98 -      }
     99 -    }
    100 -    return parts.length > 0 ? parts : null;
    101 -  }
    102 -  return null;
    103 -}
    104 -
    105 -function extractStructuredPartsFromEnvelope(
    106 -  envelope: StreamJsonUserEnvelope,
    107 -  registry?: Map<string, ToolCallRequestInfo>,
    108 -): Part[] {
    109 -  const content = envelope.message?.content;
    110 -  if (!Array.isArray(content)) {
    111 -    return [];
    112 -  }
    113 -
    114 -  const parts: Part[] = [];
    115 -  for (const block of content) {
    116 -    if (!block || typeof block !== 'object' || !('type' in block)) {
    117 -      continue;
    118 -    }
    119 -
    120 -    if (block.type === 'tool_result') {
    121 -      const toolUseId =
    122 -        typeof block.tool_use_id === 'string' && block.tool_use_id.trim()
    123 -          ? block.tool_use_id
    124 -          : envelope.parent_tool_use_id;
    125 -
    126 -      if (toolUseId && registry?.has(toolUseId)) {
    127 -        const requestInfo = registry.get(toolUseId)!;
    128 -        const responseParts = convertToFunctionResponse(
    129 -          requestInfo.name,
    130 -          requestInfo.callId,
    131 -          toolResultContentToPartList(block.content),
    132 -        );
    133 -        registry.delete(toolUseId);
    134 -        if (Array.isArray(responseParts)) {
    135 -          parts.push(...responseParts);
    136 -        } else if (responseParts) {
    137 -          parts.push(responseParts as Part);
    138 -        }
    139 -        continue;
    140 -      }
    141 -
    142 -      const fallbackText = buildToolResultFallbackText(block);
    143 -      if (fallbackText) {
    144 -        parts.push({ text: fallbackText });
    145 -      }
    146 -      continue;
    147 -    }
    148 -
    149 -    if (block.type === 'text') {
    150 -      if (block.text) {
    151 -        parts.push({ text: block.text });
    152 -      }
    153 -      continue;
    154 -    }
    155 -
    156 -    if (block.type === 'thinking') {
    157 -      if (block.thinking) {
    158 -        parts.push({ text: block.thinking });
    159 -      }
    160 -      continue;
    161 -    }
    162 -
    163 -    parts.push({ text: JSON.stringify(block) });
    164 -  }
    165 -
    166 -  return parts;
    167 -}
    168 -
    169 -function extractUsageFromGeminiClient(
    170 -  geminiClient: unknown,
    171 -): StreamJsonUsage | undefined {
    172 -  if (
    173 -    !geminiClient ||
    174 -    typeof geminiClient !== 'object' ||
    175 -    typeof (geminiClient as { getChat?: unknown }).getChat !== 'function'
    176 -  ) {
    177 -    return undefined;
    178 -  }
    179 -
    180 -  try {
    181 -    const chat = (geminiClient as { getChat: () => unknown }).getChat();
    182 -    if (
    183 -      !chat ||
    184 -      typeof chat !== 'object' ||
    185 -      typeof (chat as { getDebugResponses?: unknown }).getDebugResponses !==
    186 -        'function'
    187 -    ) {
    188 -      return undefined;
    189 -    }
    190 -
    191 -    const responses = (chat as {
    192 -      getDebugResponses: () => Array<Record<string, unknown>>;
    193 -    }).getDebugResponses();
    194 -    for (let i = responses.length - 1; i >= 0; i--) {
    195 -      const metadata = responses[i]?.['usageMetadata'] as
    196 -        | Record<string, unknown>
    197 -        | undefined;
    198 -      if (metadata) {
    199 -        const promptTokens = metadata['promptTokenCount'];
    200 -        const completionTokens = metadata['candidatesTokenCount'];
    201 -        const totalTokens = metadata['totalTokenCount'];
    202 -        const cachedTokens = metadata['cachedContentTokenCount'];
    203 -
    204 -        return {
    205 -          input_tokens:
    206 -            typeof promptTokens === 'number' ? promptTokens : undefined,
    207 -          output_tokens:
    208 -            typeof completionTokens === 'number'
    209 -              ? completionTokens
    210 -              : undefined,
    211 -          total_tokens:
    212 -            typeof totalTokens === 'number' ? totalTokens : undefined,
    213 -          cache_read_input_tokens:
    214 -            typeof cachedTokens === 'number' ? cachedTokens : undefined,
    215 -        };
    216 -      }
    217 -    }
    218 -  } catch (error) {
    219 -    console.debug('Failed to extract usage metadata:', error);
    220 -  }
    221 -
    222 -  return undefined;
    223 -}
    224 -
    225 -function calculateApproximateCost(
    226 -  usage: StreamJsonUsage | undefined,
    227 -): number | undefined {
    228 -  if (!usage) {
    229 -    return undefined;
    230 -  }
    231 -  return 0;
    232 -}
    233 -
    234 -const CONTROL_REQUEST_TIMEOUT_MS = 60_000;
    235 -const HOOK_CALLBACK_TIMEOUT_MS = CONTROL_REQUEST_TIMEOUT_MS;
    236 -
    237 -type HookCallbackMessage = {
    238 -  event: string;
    239 -  callbackId: string;
    240 -  message: string;
    241 -};
    242 -
    243 -type HookCallbackAggregateResult = {
    244 -  shouldSkip: boolean;
    245 -  shouldInterrupt: boolean;
    246 -  suppressOutput: boolean;
    247 -  messages: HookCallbackMessage[];
    248 -};
    249 -
    250 -type AwaitingApprovalToolCall = {
    251 -  status: 'awaiting_approval';
    252 -  request: ToolCallRequestInfo;
    253 -  confirmationDetails: {
    254 -    type: string;
    255 -    onConfirm: (
    256 -      outcome: ToolConfirmationOutcome,
    257 -      payload?: unknown,
    258 -    ) => Promise<void>;
    259 -  } & Record<string, unknown>;
    260 -};
    261 -
    262 -export async function runNonInteractive(
    263 -  config: Config,
    264 -  settings: LoadedSettings,
    265 -  input: string,
    266 -  prompt_id: string,
    267 -  options: RunNonInteractiveOptions = {},
    268 -): Promise<void> {
    269 -  return promptIdContext.run(prompt_id, async () => {
    270 -    const consolePatcher = new ConsolePatcher({
    271 -      stderr: true,
    272 -      debugMode: config.getDebugMode(),
    273 -    });
    274 -
    275 -    const isStreamJsonOutput = config.getOutputFormat() === 'stream-json';
    276 -    const streamJsonContext = options.streamJson;
    277 -    const streamJsonWriter = isStreamJsonOutput
    278 -      ? streamJsonContext?.writer ??
    279 -        new StreamJsonWriter(config, config.getIncludePartialMessages())
    280 -      : undefined;
    281 -    const toolCallRegistry = streamJsonContext?.toolCallRegistry;
    282 -    let totalApiDurationMs = 0;
    283 -    const startTime = Date.now();
    284 -    let turnCount = 0;
    285 -    let cleanupTemporaryOptions: (() => Promise<void>) | undefined;
    286 -
    287 -    try {
    288 -      consolePatcher.patch();
    289 -      process.stdout.on('error', (err: NodeJS.ErrnoException) => {
    290 -        if (err.code === 'EPIPE') {
    291 -          process.exit(0);
    292 -        }
    293 -      });
    294 -
    295 -      const geminiClient = config.getGeminiClient();
    296 -      const abortController = options.abortController ?? new AbortController();
    297 -      const controlContext = streamJsonContext?.controlContext;
    298 -      const pendingPermissionRequests = new Set<string>();
    299 -
    300 -      const emitHookMessages = (messages: HookCallbackMessage[]): void => {
    301 -        if (!streamJsonWriter || messages.length === 0) {
    302 -          return;
    303 -        }
    304 -        for (const { event, callbackId, message } of messages) {
    305 -          streamJsonWriter.emitSystemMessage('hook_callback', {
    306 -            event,
    307 -            callback_id: callbackId,
    308 -            message,
    309 -          });
    310 -        }
    311 -      };
    312 -
    313 -      const buildHookInput = (
    314 -        event: string,
    315 -        toolCall: ToolCallRequestInfo,
    316 -        responseSummary?: Record<string, unknown>,
    317 -      ): Record<string, unknown> => {
    318 -        const input: Record<string, unknown> = {
    319 -          event,
    320 -          tool: {
    321 -            name: toolCall.name,
    322 -            input: toolCall.args,
    323 -            call_id: toolCall.callId,
    324 -          },
    325 -        };
    326 -        if (responseSummary) {
    327 -          input['response'] = responseSummary;
    328 -        }
    329 -        return input;
    330 -      };
    331 -
    332 -      const summarizeToolResponse = (
    333 -        toolResponse: Record<string, unknown>,
    334 -      ): Record<string, unknown> => {
    335 -        const summary: Record<string, unknown> = {
    336 -          is_error: Boolean(toolResponse['error']),
    337 -        };
    338 -        if (toolResponse['resultDisplay'] !== undefined) {
    339 -          summary['result_display'] = toolResponse['resultDisplay'];
    340 -        }
    341 -        if (Array.isArray(toolResponse['responseParts'])) {
    342 -          summary['response_parts_count'] = (
    343 -            toolResponse['responseParts'] as unknown[]
    344 -          ).length;
    345 -        }
    346 -        const errorInfo = toolResponse['error'];
    347 -        if (errorInfo && typeof errorInfo === 'object') {
    348 -          const message = (errorInfo as { message?: unknown }).message;
    349 -          const type = (errorInfo as { type?: unknown }).type;
    350 -          summary['error'] = {
    351 -            message: typeof message === 'string' ? message : undefined,
    352 -            type: typeof type === 'string' ? type : undefined,
    353 -          };
    354 -        }
    355 -        return summary;
    356 -      };
    357 -
    358 -      const dispatchHookCallbacks = async (
    359 -        event: string,
    360 -        toolCall: ToolCallRequestInfo,
    361 -        responseSummary?: Record<string, unknown>,
    362 -      ): Promise<HookCallbackAggregateResult> => {
    363 -        const initial: HookCallbackAggregateResult = {
    364 -          shouldSkip: false,
    365 -          shouldInterrupt: false,
    366 -          suppressOutput: false,
    367 -          messages: [],
    368 -        };
    369 -
    370 -        if (
    371 -          !isStreamJsonOutput ||
    372 -          !streamJsonContext?.controller ||
    373 -          !controlContext ||
    374 -          controlContext.hookCallbacks.size === 0 ||
    375 -          !controlContext.registeredHookEvents?.has(event)
    376 -        ) {
    377 -          return initial;
    378 -        }
    379 -
    380 -        const callbacks: string[] = [];
    381 -        for (const [callbackId, registration] of controlContext.hookCallbacks) {
    382 -          if (registration.event === event) {
    383 -            callbacks.push(callbackId);
    384 -          }
    385 -        }
    386 -
    387 -        if (callbacks.length === 0) {
    388 -          return initial;
    389 -        }
    390 -
    391 -        const aggregate: HookCallbackAggregateResult = {
    392 -          shouldSkip: false,
    393 -          shouldInterrupt: false,
    394 -          suppressOutput: false,
    395 -          messages: [],
    396 -        };
    397 -
    398 -        for (const callbackId of callbacks) {
    399 -          try {
    400 -            const responseEnvelope = await streamJsonContext.controller.sendControlRequest(
    401 -              'hook_callback',
    402 -              {
    403 -                callback_id: callbackId,
    404 -                tool_use_id: toolCall.callId,
    405 -                input: buildHookInput(event, toolCall, responseSummary),
    406 -              },
    407 -              { timeoutMs: HOOK_CALLBACK_TIMEOUT_MS },
    408 -            );
    409 -
    410 -            if (!responseEnvelope.success) {
    411 -              const errorDetail = responseEnvelope.error;
    412 -              const errorMessage =
    413 -                typeof errorDetail === 'string'
    414 -                  ? errorDetail
    415 -                  : errorDetail && typeof errorDetail === 'object'
    416 -                    ? String((errorDetail as { message?: unknown }).message ?? '')
    417 -                    : 'Hook callback failed';
    418 -              streamJsonWriter?.emitSystemMessage('hook_callback', {
    419 -                event,
    420 -                callback_id: callbackId,
    421 -                error: errorMessage,
    422 -              });
    423 -              continue;
    424 -            }
    425 -
    426 -            const responsePayload = (
    427 -              responseEnvelope.response ?? {}
    428 -            ) as Record<string, unknown>;
    429 -
    430 -            const decisionRaw = responsePayload['decision'];
    431 -            const decision =
    432 -              typeof decisionRaw === 'string'
    433 -                ? decisionRaw.toLowerCase()
    434 -                : undefined;
    435 -            const continueFlag = responsePayload['continue'];
    436 -            if (
    437 -              decision &&
    438 -              ['cancel', 'stop', 'interrupt', 'deny', 'abort'].includes(decision)
    439 -            ) {
    440 -              aggregate.shouldSkip = true;
    441 -              if (decision === 'interrupt') {
    442 -                aggregate.shouldInterrupt = true;
    443 -              }
    444 -            }
    445 -            if (continueFlag === false) {
    446 -              aggregate.shouldSkip = true;
    447 -            }
    448 -            if (responsePayload['interrupt'] === true) {
    449 -              aggregate.shouldInterrupt = true;
    450 -            }
    451 -            if (responsePayload['suppressOutput'] === true) {
    452 -              aggregate.suppressOutput = true;
    453 -            }
    454 -
    455 -            const systemMessage = responsePayload['systemMessage'];
    456 -            if (
    457 -              typeof systemMessage === 'string' &&
    458 -              systemMessage.trim().length > 0
    459 -            ) {
    460 -              aggregate.messages.push({
    461 -                event,
    462 -                callbackId,
    463 -                message: systemMessage,
    464 -              });
    465 -            } else {
    466 -              const reason = responsePayload['reason'];
    467 -              if (typeof reason === 'string' && reason.trim().length > 0) {
    468 -                aggregate.messages.push({
    469 -                  event,
    470 -                  callbackId,
    471 -                  message: reason,
    472 -                });
    473 -              }
    474 -            }
    475 -          } catch (error) {
    476 -            streamJsonWriter?.emitSystemMessage('hook_callback', {
    477 -              event,
    478 -              callback_id: callbackId,
    479 -              error:
    480 -                error instanceof Error
    481 -                  ? error.message
    482 -                  : 'Hook callback request failed',
    483 -            });
    484 -          }
    485 -        }
    486 -
    487 -        return aggregate;
    488 -      };
    489 -
    490 -      const handleToolPermissionRequest = async (
    491 -        toolCall: AwaitingApprovalToolCall,
    492 -      ): Promise<void> => {
    493 -        const controller = streamJsonContext?.controller;
    494 -        if (!controller || !streamJsonWriter) {
    495 -          await toolCall.confirmationDetails.onConfirm(
    496 -            ToolConfirmationOutcome.ProceedOnce,
    497 -          );
    498 -          pendingPermissionRequests.delete(toolCall.request.callId);
    499 -          return;
    500 -        }
    501 -
    502 -        try {
    503 -          const responseEnvelope = await controller.sendControlRequest(
    504 -            'can_use_tool',
    505 -            {
    506 -              tool_name: toolCall.request.name,
    507 -              tool_use_id: toolCall.request.callId,
    508 -              input: toolCall.request.args,
    509 -              permission_suggestions: buildPermissionSuggestions(
    510 -                toolCall.confirmationDetails,
    511 -              ),
    512 -            },
    513 -            { timeoutMs: CONTROL_REQUEST_TIMEOUT_MS },
    514 -          );
    515 -
    516 -          if (!responseEnvelope.success) {
    517 -            const errorDetail = responseEnvelope.error;
    518 -            const errorMessage =
    519 -              typeof errorDetail === 'string'
    520 -                ? errorDetail
    521 -                : errorDetail && typeof errorDetail === 'object' && 'message' in errorDetail
    522 -                  ? String((errorDetail as { message?: unknown }).message ?? '')
    523 -                  : 'Tool permission request was rejected by control_response.';
    524 -            streamJsonWriter.emitSystemMessage('tool_permission', {
    525 -              tool: toolCall.request.name,
    526 -              behavior: 'error',
    527 -              message: errorMessage,
    528 -            });
    529 -            await toolCall.confirmationDetails.onConfirm(
    530 -              ToolConfirmationOutcome.Cancel,
    531 -            );
    532 -            return;
    533 -          }
    534 -
    535 -          const responsePayload = (
    536 -            responseEnvelope.response ?? {}
    537 -          ) as Record<string, unknown>;
    538 -          const behavior = String(responsePayload['behavior'] ?? '').toLowerCase();
    539 -          const responseMessage =
    540 -            typeof responsePayload['message'] === 'string'
    541 -              ? responsePayload['message']
    542 -              : undefined;
    543 -
    544 -          if (behavior === 'allow') {
    545 -            const updatedInput = responsePayload['updatedInput'];
    546 -            if (updatedInput && typeof updatedInput === 'object') {
    547 -              toolCall.request.args = updatedInput as Record<string, unknown>;
    548 -            }
    549 -            await toolCall.confirmationDetails.onConfirm(
    550 -              ToolConfirmationOutcome.ProceedOnce,
    551 -            );
    552 -          } else {
    553 -            await toolCall.confirmationDetails.onConfirm(
    554 -              ToolConfirmationOutcome.Cancel,
    555 -            );
    556 -          }
    557 -
    558 -          if (responseMessage) {
    559 -            streamJsonWriter.emitSystemMessage('tool_permission', {
    560 -              tool: toolCall.request.name,
    561 -              behavior,
    562 -              message: responseMessage,
    563 -            });
    564 -          }
    565 -
    566 -          if (responsePayload['interrupt']) {
    567 -            controller.interruptActiveRun();
    568 -          }
    569 -        } catch (error) {
    570 -          streamJsonWriter?.emitSystemMessage('tool_permission', {
    571 -            tool: toolCall.request.name,
    572 -            behavior: 'error',
    573 -            message:
    574 -              error instanceof Error
    575 -                ? error.message
    576 -                : 'Tool permission request failed',
    577 -          });
    578 -          await toolCall.confirmationDetails.onConfirm(
    579 -            ToolConfirmationOutcome.Cancel,
    580 -          );
    581 -        } finally {
    582 -          pendingPermissionRequests.delete(toolCall.request.callId);
    583 -        }
    584 -      };
    585 -
    586 -      const handleToolSchedulerUpdate = (toolCalls: unknown[]): void => {
    587 -        for (const call of toolCalls) {
    588 -          if (
    589 -            call &&
    590 -            typeof call === 'object' &&
    591 -            (call as { status?: string }).status === 'awaiting_approval'
    592 -          ) {
    593 -            const awaiting = call as AwaitingApprovalToolCall;
    594 -          if (!pendingPermissionRequests.has(awaiting.request.callId)) {
    595 -            pendingPermissionRequests.add(awaiting.request.callId);
    596 -            void handleToolPermissionRequest(awaiting);
    597 -          }
    598 -          }
    599 -        }
    600 -      };
    601 -
    602 -      streamJsonContext?.controller?.setActiveRunAbortController?.(
    603 -        abortController,
    604 -      );
    605 -
    606 -      cleanupTemporaryOptions = await applyTemporaryEnvelopeOptions(
    607 -        config,
    608 -        options.userEnvelope,
    609 -      );
    610 -
    611 -      let initialPartList: PartListUnion | null = null;
    612 -      let shouldEmitUserEcho = true;
    613 -
    614 -      if (options.userEnvelope) {
    615 -        const structuredParts = extractStructuredPartsFromEnvelope(
    616 -          options.userEnvelope,
    617 -          toolCallRegistry,
    618 -        );
    619 -        if (structuredParts.length > 0) {
    620 -          initialPartList = structuredParts;
    621 -          shouldEmitUserEcho = false;
    622 -        }
    623 -      }
    624 -
    625 -      if (!initialPartList) {
    626 -        let slashHandled = false;
    627 -        if (isSlashCommand(input)) {
    628 -          const slashCommandResult = await handleSlashCommand(
    629 -            input,
    630 -            abortController,
    631 -            config,
    632 -            settings,
    633 -          );
    634 -          if (slashCommandResult) {
    635 -            initialPartList = slashCommandResult as PartListUnion;
    636 -            slashHandled = true;
    637 -          }
    638 -        }
    639 -
    640 -        if (!slashHandled) {
    641 -          const { processedQuery, shouldProceed } = await handleAtCommand({
    642 -            query: input,
    643 -            config,
    644 -            addItem: (_item, _timestamp) => 0,
    645 -            onDebugMessage: () => {},
    646 -            messageId: Date.now(),
    647 -            signal: abortController.signal,
    648 -          });
    649 -
    650 -          if (!shouldProceed || !processedQuery) {
    651 -            throw new FatalInputError(
    652 -              'Exiting due to an error processing the @ command.',
    653 -            );
    654 -          }
    655 -          initialPartList = processedQuery as PartListUnion;
    656 -        }
    657 -      }
    658 -
    659 -      const initialParts = normalizePartList(initialPartList);
    660 -      let currentMessages: Content[] = [{ role: 'user', parts: initialParts }];
    661 -
    662 -      if (isStreamJsonOutput && streamJsonWriter && shouldEmitUserEcho) {
    663 -        streamJsonWriter.emitUserMessageFromParts(initialParts);
    664 -      }
    665 -
    666 -      while (true) {
    667 -        turnCount++;
    668 -        if (
    669 -          config.getMaxSessionTurns() >= 0 &&
    670 -          turnCount > config.getMaxSessionTurns()
    671 -        ) {
    672 -          handleMaxTurnsExceededError(config);
    673 -        }
    674 -        const toolCallRequests: ToolCallRequestInfo[] = [];
    675 -
    676 -        const apiStartTime = Date.now();
    677 -        const responseStream = geminiClient.sendMessageStream(
    678 -          currentMessages[0]?.parts || [],
    679 -          abortController.signal,
    680 -          prompt_id,
    681 -        );
    682 -
    683 -        const assistantBuilder = streamJsonWriter?.createAssistantBuilder();
    684 -        let responseText = '';
    685 -
    686 -        for await (const event of responseStream) {
    687 -          if (abortController.signal.aborted) {
    688 -            handleCancellationError(config);
    689 -          }
    690 -
    691 -          if (event.type === GeminiEventType.Content) {
    692 -            if (isStreamJsonOutput) {
    693 -              assistantBuilder?.appendText(event.value);
    694 -            } else if (config.getOutputFormat() === OutputFormat.JSON) {
    695 -              responseText += event.value;
    696 -            } else {
    697 -              process.stdout.write(event.value);
    698 -            }
    699 -          } else if (event.type === GeminiEventType.Thought) {
    700 -            if (isStreamJsonOutput && assistantBuilder) {
    701 -              const subject = event.value.subject?.trim();
    702 -              const description = event.value.description?.trim();
    703 -              const combined = [subject, description]
    704 -                .filter((part) => part && part.length > 0)
    705 -                .join(': ');
    706 -              if (combined.length > 0) {
    707 -                assistantBuilder.appendThinking(combined);
    708 -              }
    709 -            }
    710 -          } else if (event.type === GeminiEventType.ToolCallRequest) {
    711 -            toolCallRequests.push(event.value);
    712 -            toolCallRegistry?.set(event.value.callId, event.value);
    713 -            if (isStreamJsonOutput) {
    714 -              assistantBuilder?.appendToolUse(event.value);
    715 -            }
    716 -          }
    717 -        }
    718 -
    719 -        assistantBuilder?.finalize();
    720 -        totalApiDurationMs += Date.now() - apiStartTime;
    721 -
    722 -        if (toolCallRequests.length > 0) {
    723 -          const toolResponseParts: Part[] = [];
    724 -          pendingPermissionRequests.clear();
    725 -          const executeOptions: ExecuteToolCallOptions | undefined =
    726 -            streamJsonContext && controlContext
    727 -              ? {
    728 -                  onToolCallsUpdate: handleToolSchedulerUpdate,
    729 -                }
    730 -              : undefined;
    731 -          for (const requestInfo of toolCallRequests) {
    732 -            if (
    733 -              isStreamJsonOutput &&
    734 -              streamJsonContext?.controller &&
    735 -              controlContext
    736 -            ) {
    737 -              const hookResult = await dispatchHookCallbacks(
    738 -                'pre_tool',
    739 -                requestInfo,
    740 -              );
    741 -              emitHookMessages(hookResult.messages);
    742 -              if (hookResult.shouldInterrupt) {
    743 -                streamJsonContext.controller.interruptActiveRun();
    744 -                abortController.abort();
    745 -                return;
    746 -              }
    747 -              if (hookResult.shouldSkip) {
    748 -                toolCallRegistry?.delete(requestInfo.callId);
    749 -                continue;
    750 -              }
    751 -            }
    752 -
    753 -            const toolResponse = await executeToolCall(
    754 -              config,
    755 -              requestInfo,
    756 -              abortController.signal,
    757 -              executeOptions,
    758 -            );
    759 -
    760 -            if (toolResponse.error) {
    761 -              handleToolError(
    762 -                requestInfo.name,
    763 -                toolResponse.error,
    764 -                config,
    765 -                toolResponse.errorType || 'TOOL_EXECUTION_ERROR',
    766 -                typeof toolResponse.resultDisplay === 'string'
    767 -                  ? toolResponse.resultDisplay
    768 -                  : undefined,
    769 -              );
    770 -              if (isStreamJsonOutput) {
    771 -                const message =
    772 -                  toolResponse.resultDisplay || toolResponse.error.message;
    773 -                streamJsonWriter?.emitSystemMessage('tool_error', {
    774 -                  tool: requestInfo.name,
    775 -                  message,
    776 -                });
    777 -              }
    778 -            }
    779 -
    780 -            let suppressOutput = false;
    781 -            if (
    782 -              isStreamJsonOutput &&
    783 -              streamJsonContext?.controller &&
    784 -              controlContext
    785 -            ) {
    786 -              const responseSummary = summarizeToolResponse(
    787 -                toolResponse as unknown as Record<string, unknown>,
    788 -              );
    789 -              const postHook = await dispatchHookCallbacks(
    790 -                'post_tool',
    791 -                requestInfo,
    792 -                responseSummary,
    793 -              );
    794 -              emitHookMessages(postHook.messages);
    795 -              if (postHook.shouldInterrupt) {
    796 -                streamJsonContext.controller.interruptActiveRun();
    797 -                abortController.abort();
    798 -                return;
    799 -              }
    800 -              suppressOutput = postHook.suppressOutput;
    801 -            }
    802 -
    803 -            if (isStreamJsonOutput && !suppressOutput) {
    804 -              streamJsonWriter?.emitToolResult(requestInfo, toolResponse);
    805 -            }
    806 -
    807 -            if (toolResponse.responseParts) {
    808 -              toolResponseParts.push(...toolResponse.responseParts);
    809 -            }
    810 -
    811 -            toolCallRegistry?.delete(requestInfo.callId);
    812 -          }
    813 -          currentMessages = [{ role: 'user', parts: toolResponseParts }];
    814 -        } else {
    815 -          if (isStreamJsonOutput) {
    816 -            const usage = extractUsageFromGeminiClient(geminiClient);
    817 -            streamJsonWriter?.emitResult({
    818 -              isError: false,
    819 -              durationMs: Date.now() - startTime,
    820 -              apiDurationMs: totalApiDurationMs,
    821 -              numTurns: turnCount,
    822 -              usage,
    823 -              totalCostUsd: calculateApproximateCost(usage),
    824 -            });
    825 -          } else if (config.getOutputFormat() === OutputFormat.JSON) {
    826 -            const formatter = new JsonFormatter();
    827 -            const stats = uiTelemetryService.getMetrics();
    828 -            process.stdout.write(formatter.format(responseText, stats));
    829 -          } else {
    830 -            process.stdout.write('\n');
    831 -          }
    832 -          return;
    833 -        }
    834 -      }
    835 -    } catch (error) {
    836 -      if (isStreamJsonOutput) {
    837 -        const usage = extractUsageFromGeminiClient(config.getGeminiClient());
    838 -        const message = error instanceof Error ? error.message : String(error);
    839 -        streamJsonWriter?.emitResult({
    840 -          isError: true,
    841 -          durationMs: Date.now() - startTime,
    842 -          apiDurationMs: totalApiDurationMs,
    843 -          numTurns: turnCount,
    844 -          errorMessage: message,
    845 -          usage,
    846 -          totalCostUsd: calculateApproximateCost(usage),
    847 -        });
    848 -      }
    849 -      handleError(error, config);
    850 -    } finally {
    851 -      await cleanupTemporaryOptions?.();
    852 -      streamJsonContext?.controller?.setActiveRunAbortController?.(null);
    853 -      consolePatcher.cleanup();
    854 -      if (isTelemetrySdkInitialized()) {
    855 -        await shutdownTelemetry(config);
    856 -      }
    857 -    }
    858 -  });
    859 -}

• Added packages/cli/src/nonInteractiveCli.ts (+384 -0)
      1 +/**
      2 + * @license
      3 + * Copyright 2025 Google LLC
      4 + * SPDX-License-Identifier: Apache-2.0
      5 + */
      6 +
      7 +import type { Config, ToolCallRequestInfo } from '@qwen-code/qwen-code-core';
      8 +import { isSlashCommand } from './ui/utils/commandUtils.js';
      9 +import type { LoadedSettings } from './config/settings.js';
     10 +import {
     11 +  executeToolCall,
     12 +  shutdownTelemetry,
     13 +  isTelemetrySdkInitialized,
     14 +  GeminiEventType,
     15 +  FatalInputError,
     16 +  promptIdContext,
     17 +  OutputFormat,
     18 +  JsonFormatter,
     19 +  uiTelemetryService,
     20 +} from '@qwen-code/qwen-code-core';
     21 +import type { Content, Part, PartListUnion } from '@google/genai';
     22 +import { StreamJsonWriter } from './streamJson/writer.js';
     23 +import type {
     24 +  StreamJsonUsage,
     25 +  StreamJsonUserEnvelope,
     26 +} from './streamJson/types.js';
     27 +import type { StreamJsonController } from './streamJson/controller.js';
     28 +
     29 +import { handleSlashCommand } from './nonInteractiveCliCommands.js';
     30 +import { ConsolePatcher } from './ui/utils/ConsolePatcher.js';
     31 +import { handleAtCommand } from './ui/hooks/atCommandProcessor.js';
     32 +import {
     33 +  handleError,
     34 +  handleToolError,
     35 +  handleCancellationError,
     36 +  handleMaxTurnsExceededError,
     37 +} from './utils/errors.js';
     38 +
     39 +export interface RunNonInteractiveOptions {
     40 +  abortController?: AbortController;
     41 +  streamJson?: {
     42 +    writer?: StreamJsonWriter;
     43 +    controller?: StreamJsonController;
     44 +  };
     45 +  userEnvelope?: StreamJsonUserEnvelope;
     46 +}
     47 +
     48 +function normalizePartList(parts: PartListUnion | null): Part[] {
     49 +  if (!parts) {
     50 +    return [];
     51 +  }
     52 +
     53 +  if (typeof parts === 'string') {
     54 +    return [{ text: parts }];
     55 +  }
     56 +
     57 +  if (Array.isArray(parts)) {
     58 +    return parts.map((part) =>
     59 +      typeof part === 'string' ? { text: part } : (part as Part),
     60 +    );
     61 +  }
     62 +
     63 +  return [parts as Part];
     64 +}
     65 +
     66 +function extractPartsFromEnvelope(
     67 +  envelope: StreamJsonUserEnvelope | undefined,
     68 +): PartListUnion | null {
     69 +  if (!envelope) {
     70 +    return null;
     71 +  }
     72 +
     73 +  const content = envelope.message?.content;
     74 +  if (typeof content === 'string') {
     75 +    return content;
     76 +  }
     77 +
     78 +  if (Array.isArray(content)) {
     79 +    const parts: Part[] = [];
     80 +    for (const block of content) {
     81 +      if (!block || typeof block !== 'object' || !('type' in block)) {
     82 +        continue;
     83 +      }
     84 +      if (block.type === 'text' && block.text) {
     85 +        parts.push({ text: block.text });
     86 +      } else {
     87 +        parts.push({ text: JSON.stringify(block) });
     88 +      }
     89 +    }
     90 +    return parts.length > 0 ? parts : null;
     91 +  }
     92 +
     93 +  return null;
     94 +}
     95 +
     96 +function extractUsageFromGeminiClient(
     97 +  geminiClient: unknown,
     98 +): StreamJsonUsage | undefined {
     99 +  if (
    100 +    !geminiClient ||
    101 +    typeof geminiClient !== 'object' ||
    102 +    typeof (geminiClient as { getChat?: unknown }).getChat !== 'function'
    103 +  ) {
    104 +    return undefined;
    105 +  }
    106 +
    107 +  try {
    108 +    const chat = (geminiClient as { getChat: () => unknown }).getChat();
    109 +    if (
    110 +      !chat ||
    111 +      typeof chat !== 'object' ||
    112 +      typeof (chat as { getDebugResponses?: unknown }).getDebugResponses !==
    113 +        'function'
    114 +    ) {
    115 +      return undefined;
    116 +    }
    117 +
    118 +    const responses = (chat as {
    119 +      getDebugResponses: () => Array<Record<string, unknown>>;
    120 +    }).getDebugResponses();
    121 +    for (let i = responses.length - 1; i >= 0; i--) {
    122 +      const metadata = responses[i]?.['usageMetadata'] as
    123 +        | Record<string, unknown>
    124 +        | undefined;
    125 +      if (metadata) {
    126 +        const promptTokens = metadata['promptTokenCount'];
    127 +        const completionTokens = metadata['candidatesTokenCount'];
    128 +        const totalTokens = metadata['totalTokenCount'];
    129 +        const cachedTokens = metadata['cachedContentTokenCount'];
    130 +
    131 +        return {
    132 +          input_tokens:
    133 +            typeof promptTokens === 'number' ? promptTokens : undefined,
    134 +          output_tokens:
    135 +            typeof completionTokens === 'number'
    136 +              ? completionTokens
    137 +              : undefined,
    138 +          total_tokens:
    139 +            typeof totalTokens === 'number' ? totalTokens : undefined,
    140 +          cache_read_input_tokens:
    141 +            typeof cachedTokens === 'number' ? cachedTokens : undefined,
    142 +        };
    143 +      }
    144 +    }
    145 +  } catch (error) {
    146 +    console.debug('Failed to extract usage metadata:', error);
    147 +  }
    148 +
    149 +  return undefined;
    150 +}
    151 +
    152 +function calculateApproximateCost(
    153 +  usage: StreamJsonUsage | undefined,
    154 +): number | undefined {
    155 +  if (!usage) {
    156 +    return undefined;
    157 +  }
    158 +  return 0;
    159 +}
    160 +
    161 +export async function runNonInteractive(
    162 +  config: Config,
    163 +  settings: LoadedSettings,
    164 +  input: string,
    165 +  prompt_id: string,
    166 +  options: RunNonInteractiveOptions = {},
    167 +): Promise<void> {
    168 +  return promptIdContext.run(prompt_id, async () => {
    169 +    const consolePatcher = new ConsolePatcher({
    170 +      stderr: true,
    171 +      debugMode: config.getDebugMode(),
    172 +    });
    173 +
    174 +    const isStreamJsonOutput = config.getOutputFormat() === 'stream-json';
    175 +    const streamJsonContext = options.streamJson;
    176 +    const streamJsonWriter = isStreamJsonOutput
    177 +      ? streamJsonContext?.writer ??
    178 +        new StreamJsonWriter(config, config.getIncludePartialMessages())
    179 +      : undefined;
    180 +
    181 +    let turnCount = 0;
    182 +    let totalApiDurationMs = 0;
    183 +    const startTime = Date.now();
    184 +
    185 +    try {
    186 +      consolePatcher.patch();
    187 +      process.stdout.on('error', (err: NodeJS.ErrnoException) => {
    188 +        if (err.code === 'EPIPE') {
    189 +          process.exit(0);
    190 +        }
    191 +      });
    192 +
    193 +      const geminiClient = config.getGeminiClient();
    194 +      const abortController = options.abortController ?? new AbortController();
    195 +      streamJsonContext?.controller?.setActiveRunAbortController?.(
    196 +        abortController,
    197 +      );
    198 +
    199 +      let initialPartList: PartListUnion | null =
    200 +        extractPartsFromEnvelope(options.userEnvelope);
    201 +
    202 +      if (!initialPartList) {
    203 +        let slashHandled = false;
    204 +        if (isSlashCommand(input)) {
    205 +          const slashCommandResult = await handleSlashCommand(
    206 +            input,
    207 +            abortController,
    208 +            config,
    209 +            settings,
    210 +          );
    211 +          if (slashCommandResult) {
    212 +            initialPartList = slashCommandResult as PartListUnion;
    213 +            slashHandled = true;
    214 +          }
    215 +        }
    216 +
    217 +        if (!slashHandled) {
    218 +          const { processedQuery, shouldProceed } = await handleAtCommand({
    219 +            query: input,
    220 +            config,
    221 +            addItem: (_item, _timestamp) => 0,
    222 +            onDebugMessage: () => {},
    223 +            messageId: Date.now(),
    224 +            signal: abortController.signal,
    225 +          });
    226 +
    227 +          if (!shouldProceed || !processedQuery) {
    228 +            throw new FatalInputError(
    229 +              'Exiting due to an error processing the @ command.',
    230 +            );
    231 +          }
    232 +          initialPartList = processedQuery as PartListUnion;
    233 +        }
    234 +      }
    235 +
    236 +      if (!initialPartList) {
    237 +        initialPartList = [{ text: input }];
    238 +      }
    239 +
    240 +      const initialParts = normalizePartList(initialPartList);
    241 +      let currentMessages: Content[] = [{ role: 'user', parts: initialParts }];
    242 +
    243 +      if (streamJsonWriter) {
    244 +        streamJsonWriter.emitUserMessageFromParts(initialParts);
    245 +      }
    246 +
    247 +      while (true) {
    248 +        turnCount++;
    249 +        if (
    250 +          config.getMaxSessionTurns() >= 0 &&
    251 +          turnCount > config.getMaxSessionTurns()
    252 +        ) {
    253 +          handleMaxTurnsExceededError(config);
    254 +        }
    255 +
    256 +        const toolCallRequests: ToolCallRequestInfo[] = [];
    257 +        const apiStartTime = Date.now();
    258 +        const responseStream = geminiClient.sendMessageStream(
    259 +          currentMessages[0]?.parts || [],
    260 +          abortController.signal,
    261 +          prompt_id,
    262 +        );
    263 +
    264 +        const assistantBuilder = streamJsonWriter?.createAssistantBuilder();
    265 +        let responseText = '';
    266 +
    267 +        for await (const event of responseStream) {
    268 +          if (abortController.signal.aborted) {
    269 +            handleCancellationError(config);
    270 +          }
    271 +
    272 +          if (event.type === GeminiEventType.Content) {
    273 +            if (streamJsonWriter) {
    274 +              assistantBuilder?.appendText(event.value);
    275 +            } else if (config.getOutputFormat() === OutputFormat.JSON) {
    276 +              responseText += event.value;
    277 +            } else {
    278 +              process.stdout.write(event.value);
    279 +            }
    280 +          } else if (event.type === GeminiEventType.Thought) {
    281 +            if (streamJsonWriter) {
    282 +              const subject = event.value.subject?.trim();
    283 +              const description = event.value.description?.trim();
    284 +              const combined = [subject, description]
    285 +                .filter((part) => part && part.length > 0)
    286 +                .join(': ');
    287 +              if (combined.length > 0) {
    288 +                assistantBuilder?.appendThinking(combined);
    289 +              }
    290 +            }
    291 +          } else if (event.type === GeminiEventType.ToolCallRequest) {
    292 +            toolCallRequests.push(event.value);
    293 +            if (streamJsonWriter) {
    294 +              assistantBuilder?.appendToolUse(event.value);
    295 +            }
    296 +          }
    297 +        }
    298 +
    299 +        assistantBuilder?.finalize();
    300 +        totalApiDurationMs += Date.now() - apiStartTime;
    301 +
    302 +        if (toolCallRequests.length > 0) {
    303 +          const toolResponseParts: Part[] = [];
    304 +          for (const requestInfo of toolCallRequests) {
    305 +            const toolResponse = await executeToolCall(
    306 +              config,
    307 +              requestInfo,
    308 +              abortController.signal,
    309 +            );
    310 +
    311 +            if (toolResponse.error) {
    312 +              handleToolError(
    313 +                requestInfo.name,
    314 +                toolResponse.error,
    315 +                config,
    316 +                toolResponse.errorType || 'TOOL_EXECUTION_ERROR',
    317 +                typeof toolResponse.resultDisplay === 'string'
    318 +                  ? toolResponse.resultDisplay
    319 +                  : undefined,
    320 +              );
    321 +              if (streamJsonWriter) {
    322 +                const message =
    323 +                  toolResponse.resultDisplay || toolResponse.error.message;
    324 +                streamJsonWriter.emitSystemMessage('tool_error', {
    325 +                  tool: requestInfo.name,
    326 +                  message,
    327 +                });
    328 +              }
    329 +            }
    330 +
    331 +            if (streamJsonWriter) {
    332 +              streamJsonWriter.emitToolResult(requestInfo, toolResponse);
    333 +            }
    334 +
    335 +            if (toolResponse.responseParts) {
    336 +              toolResponseParts.push(...toolResponse.responseParts);
    337 +            }
    338 +          }
    339 +          currentMessages = [{ role: 'user', parts: toolResponseParts }];
    340 +        } else {
    341 +          if (streamJsonWriter) {
    342 +            const usage = extractUsageFromGeminiClient(geminiClient);
    343 +            streamJsonWriter.emitResult({
    344 +              isError: false,
    345 +              durationMs: Date.now() - startTime,
    346 +              apiDurationMs: totalApiDurationMs,
    347 +              numTurns: turnCount,
    348 +              usage,
    349 +              totalCostUsd: calculateApproximateCost(usage),
    350 +            });
    351 +          } else if (config.getOutputFormat() === OutputFormat.JSON) {
    352 +            const formatter = new JsonFormatter();
    353 +            const stats = uiTelemetryService.getMetrics();
    354 +            process.stdout.write(formatter.format(responseText, stats));
    355 +          } else {
    356 +            process.stdout.write('\n');
    357 +          }
    358 +          return;
    359 +        }
    360 +      }
    361 +    } catch (error) {
    365 +        streamJsonWriter.emitResult({
    366 +          isError: true,
    367 +          durationMs: Date.now() - startTime,
    368 +          apiDurationMs: totalApiDurationMs,
    369 +          numTurns: turnCount,
    370 +          errorMessage: message,
    371 +          usage,
    372 +          totalCostUsd: calculateApproximateCost(usage),
    373 +        });
    374 +      }
    375 +      handleError(error, config);
    376 +    } finally {
    377 +      streamJsonContext?.controller?.setActiveRunAbortController?.(null);
    378 +      consolePatcher.cleanup();
    379 +      if (isTelemetrySdkInitialized()) {
    380 +        await shutdownTelemetry(config);
    381 +      }
    382 +    }
    383 +  });
    384 +}

• Explored
  └ Read session.ts

• Edited packages/cli/src/streamJson/session.ts (+1 -21)
      7  import readline from 'node:readline';
      8 -import type { Config, ToolCallRequestInfo } from '@qwen-code/qwen-code-core';
      8 +import type { Config } from '@qwen-code/qwen-code-core';
      9  import {
        ⋮
     28    envelope?: StreamJsonUserEnvelope;
     29 -}
     30 -
     31 -interface HookCallbackRegistration {
     32 -  event: string;
     33 -}
     34 -
     35 -export interface StreamJsonControlContext {
     36 -  hookCallbacks: Map<string, HookCallbackRegistration>;
     37 -  registeredHookEvents: Set<string>;
     29  }
        ⋮
     42    const controller = new StreamJsonController(writer);
     52 -  const controlContext: StreamJsonControlContext = {
     53 -    hookCallbacks: new Map<string, HookCallbackRegistration>(),
     54 -    registeredHookEvents: new Set<string>(),
     43    const promptQueue: PromptJob[] = [];
        ⋮
     67        abortController,
     82 -      toolCallRegistry,
     83 -      controlContext,
    163    abortController: AbortController,
    180 -  toolCallRegistry: Map<string, ToolCallRequestInfo>,
    181 -  controlContext: StreamJsonControlContext,
    164  ): Promise<void> {
        ⋮
    180        controller,
    199 -      toolCallRegistry,
    200 -      controlContext,
    181      },

• I'm removing unused definitions and imports related to stream JSON controls, updating the runNonInteractive function
  and its options for consistency, and aligning tests with these changes by ensuring obsolete tests are removed
  and remaining ones don't break. I’ll also update imports and prepare for building and testing before refreshing
  documentation to reflect removed features.

• Ran npm run build
  └
    > @qwen-code/qwen-code@0.1.1 build
    … +37 lines
    [watch] build started
    [watch] build finished

• Ran npm run test --workspace packages/cli -- src/nonInteractiveCli.test.ts
  └
    > @qwen-code/qwen-code@0.1.1 test
    … +15 lines
    JUNIT report written to /home/kdump/llm/project/qwen-code/packages/cli/junit.xml
     % Coverage report from v8

• I'm preparing to update the staging doc task1-2-4.md with notes about the features we've deferred per PR comments.
  I'll add a new section listing removed features and adjust checklists to mark what we've completed versus what remains
• Edited ~/llm/project/qwen-code/openspec/lightweight-tasks/task1-2-4.md (+7 -0)
     6  - 2025-10-30 03:48:50: 完成阶段 3 非交互主循环迁移，实现 stream-json 事件并通过定向单测。
     7 +- 2025-10-30 04:12:00: 根据 PR 建议收敛 nonInteractiveCli 改动范围，暂缓 Hook/权限等高级逻辑。
     8  ---
       ⋮
    34     - 动作：运行定向单测并视需要补充关联套件，确保所有流式场景与错误分支均覆盖；必要时以 stream-json session 脚本
执行冒烟复核。
    35 +
    36 +## 备忘（后续阶段待恢复）
    37 +
    38 +- Hook 回调聚合、工具权限确认与 permission_suggestions 调用链待在后续迭代中重新引入。
    39 +- Stream-json envelope 中的 `tool_result` -> function_response 转换暂未实现，需要结合
`convertToFunctionResponse` 再评估最佳挂载位置。
    40 +- 临时模型覆写（`temporary_model` 选项）与运行时模型回滚能力留待后续阶段同步。

• Edited ~/llm/project/qwen-code/openspec/lightweight-tasks/task1-2-4.md (+1 -1)
    11
    12 -- [ ] 在迁移阶段遵循 PR 审核建议：
    12 +- [x] 在迁移阶段遵循 PR 审核建议：
    13    - 控制 `packages/cli/src/nonInteractiveCli.ts` 的侵入式调整，优先确保 stream-json 与 control_request/response
通路落地，再视情况拆分模块。

• Ran git status -sb
  └ ## feature/stream-json-migration...origin/feature/stream-json-migration
     M packages/cli/src/gemini.tsx
    … +7 lines
    ?? packages/cli/src/streamJson/input.ts
    ?? packages/cli/src/streamJson/session.ts

  documentation updates, and next steps, making sure to highlight the removal of hooking features and compliance with
  PR comments.

─ Worked for 9m 21s ────────────────────────────────────────────────────────────────────────────────────────────────────
    调、权限协商与临时模型覆写等侵入式逻辑，以符合 PR 审核建议。
  - streamJson/session.ts：同步收敛为轻量会话管线，仅负责解析输入、调度 runNonInteractive，暂不传递工具注册表或 Hook 上
    下文。
  - nonInteractiveCli.test.ts：删去与已剔除功能对应的用例，保留基础 stream-json 行为与 usage 统计的断言。
  - streamJson/writer.test.ts：调整解析逻辑以兼容精简后的类型使用。

  测试

  - npm run build
  - npm run test --workspace packages/cli -- src/nonInteractiveCli.test.ts

  备忘

  - Hook 回调、工具权限确认、tool_result 转换以及临时模型覆写已在执行计划中标记为后续迭代项，当前未实现。