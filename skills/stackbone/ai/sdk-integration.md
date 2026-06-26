# `stackbone.ai` — SDK integration

The official `openai` SDK with `baseURL` overridden to OpenRouter. Gives you 300+ models behind one API (Claude, GPT, Gemini, Llama, Mistral, Qwen, image models, embedding models) with a single key. The key is a **Stackbone sub-key** with a monthly limit set per organization tier; pricing is **passthrough — no markup**.

## Connection

`import { stackbone } from '@stackbone/sdk'`. Reach `stackbone.ai` from any tool's `execute()` or any workflow step — the platform injects the org's managed `OPENROUTER_API_KEY` (passthrough cost, no markup) and the SDK reads it lazily; you never instantiate `openai` yourself.

Metering happens **out-of-band** via the OpenRouter Activity API. The SDK does not write to Postgres for billing. Do not roll your own usage table — the control plane reconciles spend against OpenRouter once a minute.

## Chat completions

```ts
const { data, error } = await stackbone.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages: [
    { role: 'system', content: 'You are a contract analyst.' },
    { role: 'user', content: text },
  ],
});

if (error) throw new Error(error.message);

const reply = data.choices[0]?.message?.content ?? '';
```

Every call returns the `{ data, error }` envelope — destructure it.

### Streaming (for long completions)

Streaming lets you start consuming tokens as soon as the model emits them instead of blocking on the whole completion. There is no per-chunk channel — you consume the stream and accumulate the text inside your tool or step, then return the finished result.

```ts
const { data: stream, error } = await stackbone.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages: [...],
  stream: true,
});

if (error) throw new Error(error.message);

let reply = '';
for await (const chunk of stream) {
  const delta = chunk.choices[0]?.delta?.content;
  if (delta) reply += delta;
}
```

The initial connection error arrives via `error`; once the stream is open, mid-flight errors propagate through the iterator — wrap the `for await` in `try/catch` if you need to recover (e.g. fall back to a cheaper model on `ai_credits_exhausted`).

### Tools and structured output

```ts
const { data, error } = await stackbone.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages: [...],
  tools: [{
    type: 'function',
    function: {
      name: 'lookup_contract',
      parameters: { type: 'object', properties: { id: { type: 'string' } }, required: ['id'] },
    },
  }],
  tool_choice: 'auto',
});

const call = data.choices[0]?.message?.tool_calls?.[0];
if (call) { /* execute the tool, append result, loop */ }
```

Forward the same `messages` array on each turn — OpenRouter does not maintain server-side conversation state. For structured output, pass `response_format: { type: 'json_schema', json_schema: { name, schema, strict: true } }` — honored by every major model on OpenRouter. Still parse defensively (wrap with a Zod schema) even with `strict: true`.

## Embeddings

```ts
const { data, error } = await stackbone.ai.embeddings.create({
  model: 'openai/text-embedding-3-small',
  input: ['contract clause 1', 'contract clause 2'],
});

if (error) throw new Error(error.message);

const vectors = data.data.map((d) => d.embedding); // number[][]
```

`input` accepts a string or an array of strings. Match the model's dimensions to the `pgvector` column declared in your schema (`1536` for `text-embedding-3-small`, `3072` for `-3-large`, `1024` for Cohere v3).

## Image generation

```ts
const { data, error } = await stackbone.ai.images.generate({
  model: 'openai/dall-e-3',
  prompt: 'a contract being signed, watercolor style',
});

if (error) throw new Error(error.message);

const url = data.data[0]?.url; // base64 data URL returned by OpenRouter
const b64 = data.data[0]?.b64Json; // base64 payload extracted from the data URL
```

OpenRouter implements image generation through `/chat/completions` with `modalities: ['image']`; the SDK normalises the response so you get an OpenAI-shaped `{ data: [{ url, b64Json, mimeType }] }`. If the model picks not to return an image (text-only reply), `error.code === 'ai_no_image_generated'`.

## Model list

```ts
const { data, error } = await stackbone.ai.models.list();

if (error) throw new Error(error.message);

for (const m of data.data) {
  // m.id, m.pricing, m.context_length, m.architecture, m.supported_parameters
}
```

The SDK preserves OpenRouter's extended fields (`pricing`, `context_length`, `supported_parameters`) that the upstream `openai.models.list()` would discard. Useful for building dynamic model pickers.

## Popular models

| Model ID                            | Strength                          | Notes                                         |
| ----------------------------------- | --------------------------------- | --------------------------------------------- |
| `anthropic/claude-sonnet-4.5`       | Reasoning, long context, tool use | Default for production agents. 200K context.  |
| `anthropic/claude-haiku-4.5`        | Fast, cheap, decent quality       | Good for routing / triage / extraction.       |
| `openai/gpt-4o`                     | Multimodal, fast tool use         | Solid generalist; cheaper than Claude Sonnet. |
| `openai/gpt-4o-mini`                | Cheapest 4o-tier                  | Default for tests and dev iteration.          |
| `google/gemini-2.5-pro`             | 1M context window                 | When you need to fit a whole codebase / book. |
| `meta-llama/llama-3.3-70b-instruct` | OSS, low cost                     | Fallback when proprietary models throttle.    |
| `openai/text-embedding-3-small`     | 1536-dim embeddings               | Default for `stackbone.rag`.                  |
| `openai/text-embedding-3-large`     | 3072-dim embeddings               | When recall matters more than storage cost.   |

OpenRouter slugs are stable; the canonical list is `stackbone.ai.models.list()`.

## Patterns

### Cheap model first, escalate on confidence

```ts
const draft = await stackbone.ai.chat.completions.create({
  model: 'openai/gpt-4o-mini',
  messages: [...],
  response_format: { type: 'json_schema', json_schema: { name: 'classification', schema, strict: true } },
});

if (draft.error) throw new Error(draft.error.message);

const { confidence } = JSON.parse(draft.data.choices[0].message.content);
if (confidence < 0.7) {
  // Re-ask with a stronger (and pricier) model
  const final = await stackbone.ai.chat.completions.create({
    model: 'anthropic/claude-sonnet-4.5',
    messages: [...],
  });
}
```

Saves ~10x cost on the easy 80% of inputs.

### Fall back when credits run dry

```ts
const result = await stackbone.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages,
});
if (result.error?.code === 'ai_credits_exhausted') {
  // Surface verbatim — do NOT silently downgrade unless the org member opts in.
  throw new Error('Your monthly LLM budget is exhausted. Top up in Studio.');
}
```

`ai_credits_exhausted` means the org's monthly cap has been reached. Surface the error so the org member can upgrade — silently swapping to a free model breaks the cost narrative the org agreed to.

## Best practices

1. **Always destructure `{ data, error }`.** Stream errors arrive via the iterator, but the initial connection error arrives via `error`.
2. **Stream long chats.** Pass `stream: true`, consume the stream and accumulate inside your tool or step, then return the result.
3. **Pin the model in code.** Silent model swaps make billing unpredictable — only read the model from elsewhere if the org member is explicitly choosing it.
4. **Never write usage to Postgres.** OpenRouter Activity API is the source of truth; rolling your own accounting will drift.
5. **Use `response_format` for structured output.** `{ type: 'json_schema', strict: true }` is honored by every major model on OpenRouter.
6. **Validate JSON from the model.** Even with `strict: true`, parse defensively — wrap with a Zod schema before consuming.

## Common mistakes

| Mistake                                                                   | Fix                                                                                               |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Hard-coding `process.env.OPENROUTER_API_KEY`                              | Use `stackbone.ai` — the SDK reads it.                                                            |
| Writing a `_llm_usage` table on every call                                | Don't. OpenRouter Activity API is the canonical metering pipeline.                                |
| Ignoring `error.code === 'ai_credits_exhausted'`                          | Surface it to the caller — do not silently swap to a free model.                                  |
| Calling `openai/dall-e-3` and expecting `/v1/images/generations` upstream | The SDK normalises; consume `data.data[0]` and accept `ai_no_image_generated` as a possible miss. |
| Embedding with one model, indexing with another                           | Dimensions must match the `pgvector` column. Pick one model per collection and stay there.        |

## Common error codes

| `error.code`             | Cause                                                      | Action                                                                                                                               |
| ------------------------ | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `ai_rate_limited`        | OpenRouter throttled the sub-key                           | Retry with exponential backoff; `error.retryable === true`.                                                                          |
| `ai_credits_exhausted`   | Monthly tier budget hit                                    | Surface verbatim; the org member must upgrade. Do not retry.                                                                         |
| `ai_billing_paused`      | Stripe payment failed at the org level                     | Surface verbatim; the org owner must fix payment.                                                                                    |
| `ai_moderation_blocked`  | OpenRouter's content policy refused the prompt             | Inspect `error.details.categories`; do not retry the same payload.                                                                   |
| `ai_no_image_generated`  | Model returned text instead of an image                    | Re-prompt or pick a different image model.                                                                                           |
| `ai_provider_error`      | Upstream provider 5xx (Anthropic, OpenAI, etc. downstream) | Retry with backoff; if persistent, fall back to another model.                                                                       |
| `capability_unavailable` | The negotiated protocol contract doesn't grant `ai`        | Gating is via the contract handshake, not a manifest field — surface the message; if it persists it's a control-plane/install issue. |

See the main [SKILL.md](../SKILL.md) for cross-module patterns and how `stackbone.ai` powers `stackbone.rag`'s embedding pipeline.
