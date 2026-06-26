# Background work & scheduling — `@stackbone/sdk/workflow`

There is **no separate queue system**. Durable workflows _are_ the unit of background and scheduled work: each run is checkpointed step-by-step, survives a crash, and resumes from the last completed step. You start and schedule them through the `stackbone.workflows` surface on the ambient client (`import { stackbone } from '@stackbone/sdk'`). These helpers bind when a workflow is dispatched, so call them **from inside a workflow** (a `'use workflow'` body or a `'use step'`), not from a tool.

> The shell mirror is `stackbone workflows start <name>` (see the **stackbone-cli** skill). Authoring a workflow — `'use workflow'` / `'use step'`, sibling `inputSchema` / `outputSchema` — is in the main [SKILL.md](../SKILL.md).
>
> These five helpers used to be loose named imports from `@stackbone/sdk/workflow` (`startWorkflow`, `startWorkflowAndWait`, `scheduleWorkflow`, `unschedule`, `listSchedules`). Those exports still work but are now **deprecated** — they delegate to the same implementation as `stackbone.workflows.*`. Prefer the namespaced form. The peer-bound authoring API (`requestApproval`, `defineHook`, `sleep`) stays on the `@stackbone/sdk/workflow` subpath because it eager-loads the `workflow` peer.

## Start another workflow by name

```ts
import { stackbone } from '@stackbone/sdk';

// fire-and-forget: `reconcile` runs as its OWN run; returns immediately.
const { runId } = await stackbone.workflows.start('reconcile', { invoiceId });

// durable sub-routine: suspend until `summarize` finishes, get its typed output.
const summary = await stackbone.workflows.startAndWait<Summary>('summarize', { docId });
```

- `stackbone.workflows.start(name, input)` enqueues an independent run (its own `runId`, steps, runs row) and returns `{ runId }` without waiting — use it to fan out or hand a slice off.
- `stackbone.workflows.startAndWait<T>(name, input)` blocks durably until the target reaches a terminal state, then returns its output validated against the target's `outputSchema`.
- `name` is the workflow's convention name (the `*.workflow.ts` basename), narrowed to your declared workflows once `stackbone dev` has generated `.stackbone/workflows.d.ts` — a typo is a compile error. `input` is validated against the target's `inputSchema` **before** anything is enqueued; an unresolved name or a bad shape rejects without starting a run.
- Both are durable `'use step'`s: once complete, a parent restart resumes with the memoized result and does **not** re-enqueue. A crash _while the step is mid-flight_ re-runs it, so keep targets idempotent (at-least-once).

## Recurring schedules

Two ways to run a workflow on a cron; each tick starts the named workflow as its own run.

**Declarative** (preferred for schedules that ship with the workspace) — export them next to the workflow; the build harvests them and the runtime reconciles them on deploy/boot:

```ts
// workflows/digest.workflow.ts
export const schedules = [{ cron: '0 9 * * *', input: { scope: 'all' } }];
```

**Dynamic** (registered at run time — e.g. when a user enables a sync):

```ts
import { stackbone } from '@stackbone/sdk';

await stackbone.workflows.schedule('digest', { scope: 'team-42' }, '0 9 * * *'); // idempotent by name
const active = await stackbone.workflows.listSchedules(); // declarative + dynamic
await stackbone.workflows.unschedule('digest');
```

- `stackbone.workflows.schedule(name, input, cron)` registers (or replaces) a trigger keyed by `name` — re-calling with the same `name` updates the cadence/input in place. `stackbone.workflows.unschedule(name)` removes it; `stackbone.workflows.listSchedules()` returns both declarative and dynamic schedules. Cron is standard 5-field syntax; dynamic schedules are namespaced apart so a deploy never prunes them.
- Prefer these over `setInterval` / `pg_cron` for anything that must wake the agent — an in-process timer dies with the run, and `pg_cron` can't start a workflow.

## Durable RAG ingest

`ingestDocuments(...)` is the same idea specialised for RAG — it stages content to storage in a durable step, then runs the reserved `rag-ingest` workflow:

```ts
import { ingestDocuments } from '@stackbone/sdk/workflow';

const { documentId, chunks } = await ingestDocuments({
  collection: 'docs',
  content: markdown,
  contentType: 'text/markdown',
}); // or pass { storageKey } when the original is already staged
```

See [rag/sdk-integration.md](../rag/sdk-integration.md) for the inline `stackbone.rag.ingest` / `ingestAsync` path.

## Best practices

1. **Workflows are the durable unit.** Anything beyond a single turn — long work, retries, schedules, fan-out — is a workflow, not an in-process timer.
2. **Keep targets idempotent.** Triggers are at-least-once on a mid-flight crash; design the target so a replay is safe.
3. **Validate at the boundary.** A bad `input` is rejected before a run starts — surface it, don't swallow it.
4. **Prefer declarative schedules** for cadences that ship with the workspace; reach for `stackbone.workflows.schedule(...)` only for user-driven, runtime-registered ones.

See the main [SKILL.md](../SKILL.md) for the durable-workflow authoring model.
