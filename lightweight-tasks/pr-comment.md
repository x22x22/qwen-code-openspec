# PR审核员对 qwen-code-third-party-integration-draft-discussion 分支PR部分的建议

我们当前是 feature/stream-json-migration 分支，在做 qwen-code-third-party-integration-draft-discussion 分支功能迁移的时候，请留意以下PR审核员的建议。

- packages/cli/src/nonInteractiveCli.ts

```diff
+     }
+ 
+     return aggregate;
+   };
```

```comment
This seems like a fairly intrusive change to non-interactive mode.
IMHO, it's necessary, but I'm wondering if we can retain the core stream JSON and control_request/response components first to avoid making too many changes all at once. This will also help us consider how to split them into modules with clearer semantics as we iterate.
```

- packages/cli/src/streamJson/types.ts
  
```diff
+ export type StreamJsonEnvelope =
+   | StreamJsonOutputEnvelope
+   | StreamJsonInputEnvelope;
```

```comment
Perhaps we can organize these message types as separate protocols so that future SDKs can align with them.
Although the TS version of the Claude Code Agent SDK doesn't explicitly depend on the CLI package, it has a lower-level `@anthropic-ai/sdk` as its SoT. This differs from our situation.
I'd love to hear your thoughts.
Also, maybe we could just define them as `*Message` instead of `*Evelope`?
```

- packages/cli/src/streamJson/session.ts

```diff
+     hookCallbacks: new Map<string, HookCallbackRegistration>(),
+     registeredHookEvents: new Set<string>(),
+     mcpClients: new Map<string, { client: Client; config: MCPServerConfig }>(),
+   };
```

```comment
Love the thought of ` `:)
```

- packages/cli/src/streamJson/input.ts

```diff
+ function writeEnvelope(envelope: StreamJsonOutputEnvelope): void {
+   process.stdout.write(`${serializeStreamJsonEnvelope(envelope)}\n`);
+ }
```

```comment
`input.ts` looks like a utility for the input stream.
However, `readStreamJsonInput` and its dependency `parseStreamJsonInputFromIterable` are not used.
Also, `handleControlRequest` is duplicated in session.ts.
`writeEnvelope` is already supported by `writer`.
`extractUserMessageText` should be the only valid code.

Given this, I believe we can consider combining it with `writer` to form a stream manipulation I/O module as a dependency of `session`, responsible for reading/writing, parsing/wrapping.
```

- packages/cli/src/streamJson/session.ts

```diff
+   writer: StreamJsonWriter,
+   controlContext: StreamJsonControlContext,
+ ): Promise<boolean> {
+   const subtype = envelope.request?.subtype;
```

```comment
Seeing that `session` does a lot of message routing work, just leave a questiong here in case I'm missing something: since we already have a `controller.ts` that handles control_request, can we put it there to make the implementation clearer?
```
