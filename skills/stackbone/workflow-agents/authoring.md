# Workflow → agent (the hybrid)

A **workflow-agent** is the composed pattern: a durable [workflow](../workflows/authoring.md) owns the fixed, auditable backbone and delegates an open-ended turn to a [deep agent](../agents/authoring.md). The workflow validates and records; the agent thinks (its model + tools). Scaffold the pair with `stackbone add workflow-agent <name>`, or wire a step into an existing workflow with `stackbone add workflow <name> --calls <agent>`.

From a `'use step'`, call a sibling agent **in-process** with `callDeepAgent` from `@stackbone/sdk/workflow`:

```ts
import { callDeepAgent } from '@stackbone/sdk/workflow';

async function askAgent(message: string) {
  'use step';
  const { text } = await callDeepAgent('lead-qualifier', message);
  return text;
}
```

- `callDeepAgent(name, input)` selects an agent by name (the `deep-agents/<name>/` folder basename). Once `stackbone dev` has generated `.stackbone/agents.d.ts`, the name is **typed** — a typo is a compile error.
- The call runs the agent's LangGraph graph **in the same process** — no HTTP, no signing, no session plumbing.
- The result is `{ text }` — the agent's final reply as a string. If you need structure, ask the agent for JSON in the message and parse/validate it in the step.

## Input forms

`input` is a bare string (shorthand for one user message) or a full message list to carry context:

```ts
const { text } = await callDeepAgent('lead-qualifier', {
  messages: [
    { role: 'system', content: 'Answer with a single JSON object.' },
    { role: 'user', content: `Qualify this lead: ${JSON.stringify(lead)}` },
  ],
});
```

Roles are `system` | `user` | `assistant`. For a multi-turn exchange inside one workflow, accumulate the messages yourself and pass the growing list on each call — the workflow owns the conversation state (checkpointed step by step), not the agent.

## The turn is ONE durable step

The whole agent turn — model calls, tool calls, everything — executes inside the single `'use step'` that awaits it. That means:

- The step's **checkpoint** is the returned `{ text }`; on resume the turn is not re-run.
- On **retry** (the step failed), the whole turn re-runs from zero — the agent's side-effectful tools must be **idempotent**, exactly like any other step side-effect.
- Keep post-processing (parsing, DB writes) in **separate steps** so a parse failure doesn't re-run the model call.

```ts
export async function qualifyLeadWorkflow(input: z.infer<typeof inputSchema>) {
  'use workflow';
  const reply = await askAgent(`Qualify: ${input.email}`); // step 1 — the agent turn
  const saved = await persistVerdict(input.leadId, reply); // step 2 — side effect, idempotent
  return saved;
}
```

## References

- [agents/authoring.md](../agents/authoring.md) — authoring the agent itself.
- [workflows/authoring.md](../workflows/authoring.md) — the directive rules, schemas, and streaming chat workflows (`getWritable()`).
