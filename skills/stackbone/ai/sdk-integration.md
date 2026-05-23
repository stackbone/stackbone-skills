# `client.ai` — SDK integration

The official `openai` SDK with `baseURL` overridden to OpenRouter. Gives you 300+ models behind one API (Claude, GPT, Gemini, Llama, Mistral, Qwen, image models, embedding models) with a single key. The key is a **Stackbone sub-key** with a monthly limit set per organization tier; pricing is **passthrough — no markup**.

## Connection

```ts
import { createClient } from '@stackbone/sdk';

const client = createClient(); // reads OPENROUTER_API_KEY at boot
```

The platform injects `OPENROUTER_API_KEY` and `OPENROUTER_BASE_URL` (the latter pins to `https://openrouter.ai/api/v1`, but the SDK uses it so on-prem swaps work). You never instantiate `openai` directly — go through `client.ai`.

Metering happens **out-of-band** via the OpenRouter Activity API. The SDK does not write to Postgres for billing. Do not roll your own usage table — the control plane reconciles spend against OpenRouter once a minute.

## Chat completions

```ts
const { data, error } = await client.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages: [
    { role: 'system', content: 'You are a contract analyst.' },
    { role: 'user', content: text },
  ],
});

if (error) return ctx.fail(error.code, error.message);

const reply = data.choices[0]?.message?.content ?? '';
return ctx.ok({ reply });
```

Forward `ctx.signal` so a wrapper-side abort cancels the upstream call cleanly:

```ts
const { data, error } = await client.ai.chat.completions.create(
  { model: 'openai/gpt-4o-mini', messages: [...] },
  { signal: ctx.signal },
);
```

### Streaming (default for long completions)

The platform's per-invocation deadline is around 30 seconds. Any chat that may take longer **must** stream — otherwise the wrapper kills it mid-response.

```ts
const { data: stream, error } = await client.ai.chat.completions.create(
  {
    model: 'anthropic/claude-sonnet-4.5',
    messages: [...],
    stream: true,
  },
  { signal: ctx.signal },
);

if (error) return ctx.fail(error.code, error.message);

for await (const chunk of stream) {
  const delta = chunk.choices[0]?.delta?.content;
  if (delta) ctx.send(delta); // SSE channel to the caller
}
```

Stream errors propagate to the iterator — wrap the `for await` in `try/catch` if you need to recover mid-flight (e.g. fall back to a cheaper model on `ai_credits_exhausted`).

### Tools and structured output

```ts
const { data, error } = await client.ai.chat.completions.create({
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

Forward the same `messages` array on each turn — OpenRouter does not maintain server-side conversation state.

## Embeddings

```ts
const { data, error } = await client.ai.embeddings.create({
  model: 'openai/text-embedding-3-small',
  input: ['contract clause 1', 'contract clause 2'],
});

if (error) return ctx.fail(error.code, error.message);

const vectors = data.data.map((d) => d.embedding); // number[][]
```

`input` accepts a string or an array of strings. Match the model's dimensions to the `pgvector` column declared in your schema (`1536` for `text-embedding-3-small`, `3072` for `-3-large`, `1024` for Cohere v3).

## Image generation

```ts
const { data, error } = await client.ai.images.generate({
  model: 'openai/dall-e-3',
  prompt: 'a contract being signed, watercolor style',
});

if (error) return ctx.fail(error.code, error.message);

const url = data.data[0]?.url; // hosted by OpenRouter, short TTL
const b64 = data.data[0]?.b64Json; // alternative if requested
```

OpenRouter implements image generation through `/chat/completions` with `modalities: ['image']`; the SDK normalises the response so you get an OpenAI-shaped `{ data: [{ url, b64Json, mimeType }] }`. If the model picks not to return an image (text-only reply), `error.code === 'ai_no_image_generated'`.

## Model list

```ts
const { data, error } = await client.ai.models.list();

if (error) return ctx.fail(error.code, error.message);

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
| `openai/text-embedding-3-small`     | 1536-dim embeddings               | Default for `client.rag`.                     |
| `openai/text-embedding-3-large`     | 3072-dim embeddings               | When recall matters more than storage cost.   |

OpenRouter slugs are stable; the canonical list is `client.ai.models.list()`.

## Patterns

### Cheap model first, escalate on confidence

```ts
const draft = await client.ai.chat.completions.create({
  model: 'openai/gpt-4o-mini',
  messages: [...],
  response_format: { type: 'json_schema', json_schema: { name: 'classification', schema, strict: true } },
});

if (draft.error) return ctx.fail(draft.error.code, draft.error.message);

const { confidence, label } = JSON.parse(draft.data.choices[0].message.content);
if (confidence < 0.7) {
  // Re-ask with a stronger (and pricier) model
  const final = await client.ai.chat.completions.create({
    model: 'anthropic/claude-sonnet-4.5',
    messages: [...],
  });
  // ...
}
```

Saves ~10x cost on the easy 80% of inputs.

### Fall back when credits run dry

```ts
const result = await client.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages,
});
if (result.error?.code === 'ai_credits_exhausted') {
  // Surface verbatim — do NOT silently downgrade unless the org member opts in.
  return ctx.fail(
    'ai_credits_exhausted',
    'Your monthly LLM budget is exhausted. Top up in Studio.',
  );
}
```

`ai_credits_exhausted` means the org's monthly cap has been reached. Surface the error so the org member can upgrade — silently swapping to a free model breaks the cost narrative the org agreed to.

## Best practices

1. **Always destructure `{ data, error }`.** Stream errors arrive via the iterator, but the initial connection error arrives via `error`.
2. **Stream by default for chat.** Non-streamed completions over ~30 s hit the wrapper timeout.
3. **Forward `ctx.signal`.** A user-abort or wrapper-timeout should cancel the upstream call, not leak it.
4. **Pin the model in code.** Don't read it from `client.config` unless the org member is explicitly choosing it — silent model swaps make billing unpredictable.
5. **Never write usage to Postgres.** OpenRouter Activity API is the source of truth; rolling your own accounting will drift.
6. **Use `response_format` for structured output.** `{ type: 'json_schema', strict: true }` is honored by every major model on OpenRouter.
7. **Validate JSON from the model.** Even with `strict: true`, parse defensively — wrap with a Zod schema before consuming.

## Common mistakes

| Mistake                                                                   | Fix                                                                                               |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Non-streamed call to Claude Sonnet over a long context                    | Set `stream: true` and yield deltas.                                                              |
| Hard-coding `process.env.OPENROUTER_API_KEY`                              | Use `client.ai` — the SDK reads it.                                                               |
| Writing a `_llm_usage` table on every call                                | Don't. OpenRouter Activity API is the canonical metering pipeline.                                |
| Ignoring `error.code === 'ai_credits_exhausted'`                          | Surface it to the caller — do not silently swap to a free model.                                  |
| Calling `openai/dall-e-3` and expecting `/v1/images/generations` upstream | The SDK normalises; consume `data.data[0]` and accept `ai_no_image_generated` as a possible miss. |
| Embedding with one model, indexing with another                           | Dimensions must match the `pgvector` column. Pick one model per collection and stay there.        |

## Common error codes

| `error.code`             | Cause                                                      | Action                                                                |
| ------------------------ | ---------------------------------------------------------- | --------------------------------------------------------------------- |
| `ai_rate_limited`        | OpenRouter throttled the sub-key                           | Retry with exponential backoff; `error.retryable === true`.           |
| `ai_credits_exhausted`   | Monthly tier budget hit                                    | Surface verbatim; the org member must upgrade. Do not retry.          |
| `ai_billing_paused`      | Stripe payment failed at the org level                     | Surface verbatim; the org owner must fix payment.                     |
| `ai_moderation_blocked`  | OpenRouter's content policy refused the prompt             | Inspect `error.details.categories`; do not retry the same payload.    |
| `ai_timeout`             | Upstream model took longer than the wrapper allows         | Switch to `stream: true` or to a faster model.                        |
| `ai_no_image_generated`  | Model returned text instead of an image                    | Re-prompt or pick a different image model.                            |
| `ai_provider_error`      | Upstream provider 5xx (Anthropic, OpenAI, etc. downstream) | Retry with backoff; if persistent, fall back to another model.        |
| `capability_not_granted` | `agent.yaml` does not list `ai` in `capabilities`          | Declare `capabilities: [ai]` — see [agent-yaml.md](../agent-yaml.md). |

See the main [SKILL.md](../SKILL.md) for cross-module patterns and how `client.ai` powers `client.rag`'s embedding pipeline.
