# Interview: building a **workflow**

A workflow is durable, replayable code: a plain async function marked `'use workflow'` that takes a validated **input**, runs a series of `'use step'` checkpoints, and returns a validated **output**. No conversation. You scaffolded it with `stackbone init <name> --with workflow` or `stackbone add workflow <name>`, which writes `workflows/<name>.workflow.ts` with the sibling-export contract already stubbed:

```ts
export const inputSchema = z.object({ /* … */ });
export const outputSchema = z.object({ /* … */ });
export async function <name>Workflow(input: z.infer<typeof inputSchema>) {
  'use workflow';
  // steps…
}
```

Your job here is to nail the **input schema**, the **output schema**, and the **steps**. For the exact directive rules and trigger paths follow the **stackbone** skill → `workflows/authoring.md`.

## The interview — ask one at a time

1. **Trigger.** "How does this workflow get kicked off — a user/Studio `start`, a schedule (cron), another workflow, or a RAG ingest?" This tells you whether you also need the scheduling capability later.
2. **Input data → `inputSchema`.** "What data does it receive to do its job?" Turn each field into a Zod field with a `.describe()`. This is THE public contract: Studio renders an input form from it and the emulator validates every `start` payload against it. The input parameter type is `z.infer<typeof inputSchema>` — never declare a second type.
3. **Output data → `outputSchema`.** "When it finishes, what does it return?" Declare `outputSchema` **explicitly** (don't lean on the TS return type — that loses fields).
4. **The steps.** "Walk me through what it does, step by step." Each distinct side-effect (a DB write, an HTTP/connector call, an LLM call, calling an agent) becomes a `'use step'` helper. For each step confirm:
   - **What it does & returns** — its return value is the durable checkpoint.
   - **Is it idempotent?** A step is retried on failure and replayed on resume — running it twice must be safe (key writes on a stable id). Flag any step that isn't and design it to be.
   - Keep the `'use workflow'` body as **cheap deterministic glue** — it replays; do all I/O inside steps.

## Documentation

- [Workflow SDK](https://workflow-sdk.dev/v5/docs)

## Map answers → the file

| Answer                       | Lands in                                                                       |
| ---------------------------- | ------------------------------------------------------------------------------ |
| input fields                 | `export const inputSchema = z.object({ … })`                                   |
| output fields                | `export const outputSchema = z.object({ … })`                                  |
| each side-effect             | a `'use step'` helper function, called in order from the `'use workflow'` body |
| how it's addressed/triggered | the convention name = the `<name>` in `workflows/<name>.workflow.ts`           |

## Workflow-only capabilities to keep in mind

These surfaces exist **only** inside a workflow (not in an agent tool), so raise them during the checklist:

- **Human-in-the-loop** — `requestApproval()` from `@stackbone/sdk/workflow`, called from the workflow **body** (never inside a step). Pauses the run durably until a human decides.
- **Trigger / schedule other workflows** — `stackbone.workflows.start / startAndWait / schedule`, from the workflow body.
- **Call an agent** — `stackbone.agent(id).session().send(...)` from a step (this turns it into a workflow-agent → see [workflow-agent.md](workflow-agent.md)).

## Then → capabilities

Run **[capabilities.md](capabilities.md)** to decide which surfaces the steps need (database, storage, AI, RAG, connections, prompts, config, secrets, HITL, scheduling, calling an agent). Finish by booting `stackbone dev` and starting a run (`stackbone workflows start <name>` / `POST /api/workflows/<name>/start`) to verify input→output.
