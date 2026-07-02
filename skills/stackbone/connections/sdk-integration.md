# `stackbone.connection(id)` — SDK integration

`import { stackbone } from '@stackbone/sdk'`. Reach `stackbone.connection(id)` from any tool's `execute()` or any workflow step — the credential is brokered by Stackbone Connect; you only pass the operation arguments. `connection(id)` selects a third-party connector account (Slack, Gmail, Outlook, …) by its verbatim id and runs an **operation** on it. The call POSTs to the agent-facing broker, which resolves the credential, runs one provider action, and returns its output. **No token, api key or OAuth secret ever enters the agent container.**

> **Discover ids in the CLI, not at runtime.** Run `stackbone connectors --json` to see which connectors exist, the operations each exposes, and the **id or unique name** of every connection — that's the value you pass to `connection(...)`. See the **stackbone-cli** skill (`connectors` reference).

## Calling an operation

Two call forms select the same broker path:

- **Typed form** — `connection(id).<operation>(args)`. Available once `stackbone dev` generates `.stackbone/connect.d.ts` (git-ignored, regenerated when the catalog changes), which maps each connector → its operations → that operation's `args` type, derived from the introspected JSON Schema. With no generated types you get only `.call`.
- **Dynamic escape hatch** — `connection(id).call('<operation>', args, opts?)`. Always available; use it for an operation id that is not a JS identifier (e.g. a dotted `'chat.postMessage'`) or a connector with no generated types.

```ts
// typed (after `stackbone dev` generated the types):
const channels = await stackbone.connection('slack').conversationsList();

// dynamic escape hatch (dotted operation id):
const result = await stackbone.connection('slack').call('chat.postMessage', {
  channel: input.channel,
  text: input.text,
});
```

The call returns the operation **output** (the JSON the provider returned) verbatim. `args` is forwarded untouched and validated broker-side against that operation's input schema — a mismatch throws `invalid_args`. By default the call is made as `{ type: 'app' }` (the agent's own service account); pass `opts.principal` to scope it to an end user or an M2M mode.

## Disambiguating when there are several connections

When the account holds **more than one** connection of the same connector, a bare call throws with code `ambiguous`. Pass the connection **id** or **unique name** (both from `stackbone connectors --json`) to pick one:

```ts
// by unique name (readable, survives a re-connect):
await stackbone.connection('Support bot').call('chat.postMessage', args);

// by id (unambiguous):
await stackbone.connection('3f2a1b2c-89ab-4cde-8123-456789abcdef').call('chat.postMessage', args);
```

With a single connection for a connector, the sole one resolves automatically.

## Error handling

A failed call **throws** a `ConnectorCallError` — it does **not** return a `{ data, error }` envelope. Match `err.code` against the broker taxonomy; **never `instanceof`** (under bundling the SDK and your code can hold different class identities, so the code string is the only stable signal):

```ts
try {
  await stackbone.connection('slack').call('chat.postMessage', args);
} catch (err) {
  const code = (err as { code?: string }).code;
  if (code === 'invalid_args') {
    // bad arguments — fix the call shape, don't retry
  }
  throw err; // otherwise propagate
}
```

| `err.code`         | Cause                                                             | Action                                                |
| ------------------ | ----------------------------------------------------------------- | ----------------------------------------------------- |
| `invalid_args`     | `args` failed the operation's input schema, or a bad operation id | Check the call against `stackbone connectors --json`. |
| `credential_error` | Broker rejected the call (token revoked/expired, bad status)      | Surface the message; a re-connect may be needed.      |
| `timeout`          | The provider call timed out                                       | Transient — retry once.                               |
| `ambiguous`        | Several connections matched and none was selected                 | Pass an id or unique name (see disambiguation above). |
| `unauthorized`     | Caller identity rejected by the broker                            | A platform/install issue; surface and retry.          |
| `unavailable`      | Broker or provider unreachable / returned 5xx                     | Transient — surface the message and retry once.       |

## Handing an operation to the model — `connectorTool`

To let a **deep agent decide** when to run a connector operation, wrap it as a tool with `connectorTool` from `@stackbone/sdk/deep` and put it in the agent's `tools` array:

```ts
import { defineDeepAgent, connectorTool } from '@stackbone/sdk/deep';

export default defineDeepAgent({
  model: 'anthropic/claude-haiku-4.5',
  systemPrompt: '…',
  tools: [
    connectorTool({
      connector: 'slack', // connector id from `stackbone connectors --json`
      operation: 'chat.postMessage', // operation id (dotted ids are fine)
      // name?: defaults to a sanitized `slack_chat_postMessage`
      // description?: what the model reads to decide when to call it
      // schema?: a Zod schema for the args
    }),
  ],
});
```

At runtime the tool body goes through the **same broker path** as `stackbone.connection(id)` — same credential brokering, same error taxonomy — and returns the operation output to the model as a JSON string. Gate it behind human approval with `interruptOn: { slack_chat_postMessage: true }` (see [hitl/sdk-integration.md](../hitl/sdk-integration.md)).

## Authoring a connection with `connect()`

To author a richer connection (e.g. an OpenAPI connection whose auth is brokered), use `connect(connectorId)` from `@stackbone/sdk/connect`. It returns a connection-auth adapter whose `getToken` fetches a short-lived, install-scoped bearer from the broker — so the calling code never holds the raw secret:

```ts
import { connect } from '@stackbone/sdk/connect';

const auth = connect('my-openapi-connector'); // connectorId registered in Studio
```

See the wiki page on the Connect broker model (`/docs/concepts/connect`) for how identity, tokens and the operation catalog fit together.

## Best practices

1. **Match `err.code`, never `instanceof`.** A failed call (revoked token, ambiguous selector) is a real outcome — branch on the code string and propagate the rest.
2. **Prefer the unique name over the id.** It's readable and survives a re-connect; fall back to the id only when names collide.
3. **Discover, don't hardcode.** Get connection ids/names from `stackbone connectors --json`; a hardcoded id breaks when the customer re-connects.
4. **Keep `args` to the operation's schema.** It's validated broker-side — a mismatch is `invalid_args`, not a silent no-op.

## Common mistakes

| Mistake                                                      | Fix                                                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| Expecting `{ data, error }` from a call                      | It returns the output and **throws** `ConnectorCallError` — wrap it in `try/catch`.        |
| Matching the error with `instanceof`                         | Match `err.code` against the broker taxonomy; class identity is not stable under bundling. |
| Calling a connector with several connections and no selector | It throws `ambiguous`; pass an id or unique name from `stackbone connectors --json`.       |
| Hardcoding a connection id in agent code                     | Resolve it from `stackbone connectors --json`; ids change on re-connect.                   |
| Passing `args` that don't match the operation schema         | The broker rejects with `invalid_args`; check the operation's inputs in the catalog.       |

See the main [SKILL.md](../SKILL.md) for cross-module patterns, and the **stackbone-cli** skill's `connectors` reference for the discovery command.
