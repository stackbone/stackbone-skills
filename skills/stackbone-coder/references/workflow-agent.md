# Interview: building a **workflow-agent**

A workflow-agent is the composed shape: a **durable workflow** whose steps include a call to a **deep agent**. Use it when you need deterministic orchestration _and_ a reasoning agent in the loop (e.g. a pipeline that asks an agent to draft a reply, then a deterministic step persists it). You scaffolded it with `stackbone init <name> --with workflow-agent` or `stackbone add workflow-agent <name>` — which writes **both** the agent file and a workflow wired to call it (equivalent to `add workflow --calls <agent>`). `add` is offline; only `init` needs a signed-in session.

For the call shape follow the **stackbone** skill; for the full pattern read `Getting started` under SDK › Workflow + Agents and `Calling a sibling agent` under SDK › Agents (`list_docs`, then `get_doc`).

## The interview — run both, agent first

1. **The agent half.** Run the agent interview in **[agent.md](agent.md)** — role, instruction, tools, model. Here the agent is usually called by the workflow rather than by a human, so its instruction should make the _structured task_ it's asked to do crystal clear. That instruction is a prompt in the catalogue, not a string in the agent's file.
2. **The workflow half.** Run the workflow interview in **[workflow.md](workflow.md)** — `inputSchema`, `outputSchema`, the steps.
3. **The hand-off — the heart of this shape.** For the step that calls the agent, ask:
   - **What does the workflow ask the agent?** → the message (or message list) you build in that step from the workflow input.
   - **What answer does it need back?** → `callDeepAgent` returns `{ text }`; if the workflow needs structure, the agent's instruction must demand JSON and the step parses/validates it (a separate step, so a parse failure doesn't re-run the model call). Note that the demand lives in the published prompt, so an operator can weaken it from Studio without a code change — say so when you hand the project over.
   - **What does the workflow do with that answer?** → the following deterministic `'use step'` (persist, branch, return).

## The wiring (the one piece unique to this shape)

The agent call lives in a `'use step'` and goes through `callDeepAgent` from `@stackbone/sdk/workflow` — in-process, no HTTP:

```ts
import { callDeepAgent } from '@stackbone/sdk/workflow';

async function askAgent(input: z.infer<typeof inputSchema>) {
  'use step';
  const { text } = await callDeepAgent('<agent-name>', `…built from input…`);
  return text;
}
```

> The whole agent turn runs inside that one durable step: its checkpoint is the returned `{ text }`, and a retry re-runs the **whole turn** — so the agent's side-effectful tools must be idempotent. `callDeepAgent` throws on failure, which fails the step (and shows in the run's log frames). The scaffold writes `streamDeepAgent` (same import): the reply streams onto the run's chat surface and Studio serves the workflow as a chat. Swap to `callDeepAgent` when the workflow only needs the final text.

## Documentation

- `Overview` and `Getting started` under SDK › Workflow + Agents; a worked example under Examples › Workflow + Agents
- [deepagents (JS)](https://github.com/langchain-ai/deepagentsjs)
- [Workflow SDK](https://workflow-sdk.dev/docs)

## Then → capabilities

Run **[capabilities.md](capabilities.md)** for **both** halves: the agent's tools and the workflow's steps may each need different surfaces. Remember the split — `requestApproval` and `stackbone.workflows.*` live in the workflow body; the agent's tools use the data/AI/connector surfaces. Finish by booting `stackbone dev`, starting a run, and confirming the workflow drives the agent and returns the validated output.
