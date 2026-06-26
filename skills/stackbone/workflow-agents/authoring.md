# Workflow → agent (the hybrid)

A **workflow-agent** is the composed pattern: a durable [workflow](../workflows/authoring.md) owns the fixed, auditable backbone and delegates an open-ended turn to a durable [eve agent](../agents/authoring.md). The workflow validates and records; the agent thinks (its own session memory + tools). Scaffold the pair with `stackbone add workflow-agent <name>`, or wire a step into an existing workflow with `stackbone add workflow <name> --calls <agent>`.

From a `'use step'`, reach a sibling agent through the ambient `stackbone` client:

```ts
const session = stackbone.agent('lead-qualifier').session();
const response = await session.send<{ reply: string }>({
  message,
  outputSchema: z.object({ reply: z.string() }),
});
```

- `stackbone.agent(id)` selects an agent by name (the `agents/<id>/` folder basename).
- `.session(state?)` opens a session; pass a prior `state` (or continuation token) to **resume a multi-turn conversation**.
- `.send<TOutput>(input)` sends a turn. `input` is a bare string (shorthand for `{ message }`) or `{ message, outputSchema, … }`; the `outputSchema` (Zod) validates the structured reply and types `TOutput`.

The returned `response` is consumed **one of two ways** — it is a single-use stream, so pick one.

## Direct — collect the structured result

The durable-step default. Await `.result()`, get `{ data, status }`, gate on `status`, and return the data as the step's checkpoint.

```ts
import { z, stackbone } from '@stackbone/sdk';

async function askAgent(message: string) {
  'use step';
  const session = stackbone.agent('lead-qualifier').session();
  const response = await session.send<{ reply: string }>({
    message,
    outputSchema: z.object({ reply: z.string() }),
  });

  const result = await response.result(); // { data, status }
  if (result.status !== 'completed') throw new Error(`agent turn ${result.status}`);
  return result.data?.reply ?? '';
}
```

`status` is `'completed' | 'failed' | 'waiting'`; `data` is typed by the `outputSchema`. This is the shape `stackbone add workflow-agent` scaffolds.

## Streaming — forward the agent's tokens to the caller

For a **chat-style workflow** (served over `POST /api/workflows/:name/chat` as SSE), iterate the response and forward each frame to the run's streamed output with `getWritable()` from the `workflow` package.

```ts
import { stackbone } from '@stackbone/sdk';
import { getWritable } from 'workflow';

async function streamAgent(message: string) {
  'use step';
  const session = stackbone.agent('lead-qualifier').session();
  const response = await session.send({ message });

  // Forward each eve frame to the run's readable side; /chat serves it as SSE.
  const writer = getWritable().getWriter();
  try {
    for await (const event of response) {
      await writer.write(event); // event: { type, data } — text deltas, tool calls, …
    }
  } finally {
    writer.releaseLock(); // the runtime owns the stream's close (when the run ends)
  }
}
```

- Each `event` is an eve stream frame `{ type, data }` (text deltas, tool calls, …).
- A workflow only counts as **streaming** if it both imports `getWritable` from `'workflow'` **and** calls it — that build-time signal is what routes the workflow to the streaming chat panel instead of the one-shot run panel.
- Use `.result()` **or** `for await` on a given response, never both — they drain the same stream.

## Multi-turn: persist and resume the session

Read `session.state` after a turn and pass it back to `.session(state)` next time so a returning user continues the same conversation:

```ts
const session = stackbone.agent('support').session(priorState); // priorState?: state | continuation token
await (await session.send({ message })).result();
const nextState = session.state; // persist this; resume from it on the next turn
```

## References

- [Sessions, runs & streaming](https://eve.dev/docs/concepts/sessions-runs-and-streaming) — eve's turn + stream model.
- [Workflow SDK — streaming](https://workflow-sdk.dev/docs/ai) — the `getWritable()` "step that streams" pattern.
- [agents/authoring.md](../agents/authoring.md) · [workflows/authoring.md](../workflows/authoring.md)
