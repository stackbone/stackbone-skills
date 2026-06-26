# Interview: building a **workflow-agent**

A workflow-agent is the composed shape: a **durable workflow** whose steps include a call to an **eve agent**. Use it when you need deterministic orchestration _and_ a reasoning agent in the loop (e.g. a pipeline that asks an agent to draft a reply, then a deterministic step persists it). You scaffolded it with `stackbone init <name> --with workflow-agent` or `stackbone add workflow-agent <name>` — which writes **both** an agent folder and a workflow wired to call it (equivalent to `add workflow --calls <agent>`). Both register the agent eagerly, so the user must be signed in (`stackbone login`).

For the call shape (structured `result()` vs. streaming `for await`) follow the **stackbone** skill → `workflow-agents/authoring.md`.

## The interview — run both, agent first

1. **The agent half.** Run the agent interview in **[agent.md](agent.md)** — role, system prompt, tools, model. Here the agent is usually called by the workflow rather than by a human, so its `instructions.md` should make the _structured task_ it's asked to do crystal clear.
2. **The workflow half.** Run the workflow interview in **[workflow.md](workflow.md)** — `inputSchema`, `outputSchema`, the steps.
3. **The hand-off — the heart of this shape.** For the step that calls the agent, ask:
   - **What does the workflow ask the agent?** → the `message` you build in that step from the workflow input.
   - **What structured answer does it need back?** → the `outputSchema` (a `z.object`) you pass to `session().send(...)`; the agent's reply is validated against it.
   - **What does the workflow do with that answer?** → the following deterministic `'use step'` (persist, branch, return).

## The wiring (the one piece unique to this shape)

The agent call lives in a `'use step'` and goes through the ambient client — `stackbone.agent(id)`, where `id` is the agent folder basename:

```ts
async function askAgent(input: z.infer<typeof inputSchema>) {
  'use step';
  const session = stackbone.agent('<agent-name>').session();
  const response = await session.send<{ /* shape of outputSchema */ }>({
    message: `…built from input…`,
    outputSchema: z.object({ /* the structured answer you need */ }),
  });
  const result = await response.result(); // { data, status }
  return result.data ?? /* a safe fallback */;
}
```

> `result()` returns `{ data, status }` with `status` of `'completed' | 'failed' | 'waiting'`. Handle a missing `data` with a fallback so the workflow stays durable. To stream the reply instead, iterate the response with `for await…of` (see `workflow-agents/authoring.md`).

## Documentation

- [eve](https://eve.dev/docs)
- [workflow-sdk](https://workflow-sdk.dev/v5/docs)

## Then → capabilities

Run **[capabilities.md](capabilities.md)** for **both** halves: the agent's tools and the workflow's steps may each need different surfaces. Remember the split — `requestApproval` and `stackbone.workflows.*` live in the workflow body; the agent's tools use the data/AI/connector surfaces. Finish by booting `stackbone dev`, starting a run, and confirming the workflow drives the agent and returns the validated output.
