# Authoring a durable workflow

A **workflow** is durable, replayable code: a plain async function in `workflows/<name>.workflow.ts` marked `'use workflow'`, whose side-effects live in helpers marked `'use step'`. Each step is a checkpoint — kill the runtime mid-run and it resumes from the last completed step. Scaffold one with `stackbone add workflow <name>` (see the **stackbone-cli** skill).

Durability comes from the upstream [Workflow SDK](https://workflow-sdk.dev/docs) (the `workflow` package — the engine behind [Vercel Workflows](https://vercel.com/docs/workflows)). Stackbone runs it on a per-install Redis-backed runtime, so you author plain TypeScript and the runtime handles persistence, retries and replay.

## The two directives

| Directive        | Where                               | Meaning                                                                                                  |
| ---------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `'use workflow'` | first line of the workflow function | Durable orchestrator. **Replayed** from an event log on every resume — keep it deterministic glue.       |
| `'use step'`     | first line of a helper              | Runs **once**, result is persisted, **retried on failure**. Put every side-effect here. Stay idempotent. |

The body replays on resume; a completed step returns its recorded result instead of running again. So the body must be deterministic — **no `Date.now()`, no `Math.random()`, no direct I/O** — derive those inside a step. See the [directives guide](https://workflow-sdk.dev/docs/how-it-works/understanding-directives).

## Shape

```ts
// workflows/onboarding.workflow.ts
import { z, stackbone } from '@stackbone/sdk';

// THE contract — sibling exports next to the bare function. The build harvests
// them into the manifest + a live validator. Derive the input type with z.infer;
// declare outputSchema explicitly (the TS return type would lose fields).
export const inputSchema = z.object({ userId: z.string(), plan: z.enum(['free', 'pro']) });
export const outputSchema = z.object({ userId: z.string(), welcomed: z.boolean() });

export async function onboardingWorkflow(input: z.infer<typeof inputSchema>) {
  'use workflow';
  const signup = await validateSignup(input); // step 1 — deterministic
  return await persistWelcome(signup.userId); // step 2 — side effect (idempotent)
}

async function validateSignup(input: z.infer<typeof inputSchema>) {
  'use step';
  if (!input.userId) throw new Error('missing userId');
  return { userId: input.userId };
}

async function persistWelcome(userId: string) {
  'use step'; // idempotent on userId — a retry/replay must be safe
  await stackbone.database.insert(welcomes).values({ userId }).onConflictDoNothing();
  return { userId, welcomed: true };
}
```

## Discovery by convention

No registration — the workflow is discovered from the file on disk:

- The **workflow name** is the file basename without `.workflow.ts`.
- The **exported function** is `<camelCase(name)>Workflow` — e.g. `qualify-lead.workflow.ts` exports `qualifyLeadWorkflow`.

`stackbone dev` and `stackbone publish` both read this. A `stackbone.config.ts` (`defineWorkspace`) is an optional override (see [agent-yaml.md](../agent-yaml.md)).

## Serialized (FIFO) execution

By default a workflow's runs execute **concurrently** — trigger it N times and N runs go in parallel. Mark a workflow **`serial`** to run its executions **one at a time, in FIFO order**: while a run is active, every new trigger is enqueued on a durable per-workflow waiting list (each carrying its own input) and starts automatically, in arrival order, the moment the active run reaches a **terminal** state (`completed`, `failed`, or `cancelled` — a failed/cancelled run releases the lock, so it never wedges the queue).

`executionMode` is a **declared config field**, NOT a sibling export like `inputSchema`/`schedules`. Set it on the workflow's `stackbone.config.ts` entry:

```ts
// stackbone.config.ts
import { defineWorkspace } from '@stackbone/sdk';

export default defineWorkspace({
  agents: [],
  workflows: [
    // executionMode: 'serial' | 'concurrent' (default). Absence → concurrent.
    {
      name: 'reconcile',
      module: 'workflows/reconcile.workflow.ts',
      export: 'reconcileWorkflow',
      executionMode: 'serial',
    },
    { name: 'digest', module: 'workflows/digest.workflow.ts', export: 'digestWorkflow' }, // stays concurrent
  ],
});
```

- **Adding a config makes its `workflows` list REPLACE the convention scan** (explicit beats implicit). So list **every** workflow you want hosted, not only the serial one — an omitted workflow disappears from discovery. The `.workflow.ts` files themselves are unchanged.
- A queued trigger surfaces read-only as a run with status **`waiting`** in `stackbone dev` and the Studio runs view, so a backed-up queue is visible, not silent.
- Scope is **global per workflow name**, FIFO, no de-dup and no numeric limit (per-key/per-tenant concurrency and `concurrency: N` are out of scope; the same field can host them later).
- The gate lives in the shared World layer, so behavior is identical across `stackbone dev`, cloud, and self-host.

## Typed contract

Sibling `inputSchema` / `outputSchema` Zod exports are the workflow's public contract. Input is validated **before a run starts** — a bad payload is rejected with field-level issues and nothing executes. Derive the parameter type with `z.infer<typeof inputSchema>`. Inspect a declared contract with `stackbone workflows schema <name>`.

## Pauses: sleep, hooks, approval

A durable run can pause for minutes to months without holding a process open. From `@stackbone/sdk/workflow`, **called from the workflow body, never inside a `'use step'`** (a pause suspends the run, which a step may not do):

- `sleep(duration)` — pause for a fixed delay (`'30s'`, `'24h'`, or ms).
- `defineHook(...)` — pause until an external event resumes the run by token.
- `requestApproval({ token, topic, payload, timeout, fallback })` — the human-in-the-loop helper (see [hitl/sdk-integration.md](../hitl/sdk-integration.md)).

## Triggering & scheduling

- **CLI**: `stackbone workflows start <name> --input '<json>'`.
- **From another workflow**: `stackbone.workflows.start(name, input)` (fire-and-forget → `{ status: 'started', runId }`) / `stackbone.workflows.startAndWait(name, input)` (durable wait → output) from the ambient `stackbone` client. Triggering a **`serial`** target whose lock is held returns `{ status: 'queued' }` with **no `runId`** — the run is on the waiting list and starts automatically when the lock frees; `startAndWait` waits through the queue.
- **On a schedule**: a declarative `export const schedules = [{ cron, input }]` next to the workflow, or imperative `stackbone.workflows.schedule(name, input, cron)`. See [scheduling/sdk-integration.md](../scheduling/sdk-integration.md).
- **HTTP**: `POST /api/workflows/:name/start` → `{ workflowName, runId }`, or `/api/workflows/:name/chat` (SSE, for chat-style workflows).

A started workflow is an ordinary run — observe it with `stackbone runs get <id>` + `stackbone logs tail --run <id>`.

## References

- [Workflow SDK docs](https://workflow-sdk.dev/docs) · [Vercel Workflows](https://vercel.com/docs/workflows) · [directives](https://workflow-sdk.dev/docs/how-it-works/understanding-directives)
- Delegating an open-ended turn to an agent: [workflow-agents/authoring.md](../workflow-agents/authoring.md).
