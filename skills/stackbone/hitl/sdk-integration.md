# Human-in-the-loop (`requestApproval`) — SDK integration

> Folder note: this lives under `hitl/` because the **feature** is human-in-the-loop. The authoring primitive is `requestApproval()` from `@stackbone/sdk/workflow`; the lower-level inbox surface it writes to is `stackbone.approval`.

A workflow that needs a person to decide **pauses durably** with `requestApproval()` from `@stackbone/sdk/workflow` (NOT the main barrel — the subpath statically imports the `workflow` peer). It writes a row the Studio HITL inbox shows, then races the human decision against a timeout. The pause is durable: the process can die and the run resumes from the same gate when the decision (or the timeout) arrives — the hook state lives in Redis, keyed by your `token`.

**It must be called from the workflow body, never inside a `'use step'`** — pausing on a hook is a workflow primitive that suspends the run. Do all I/O in steps; keep the gate in the body.

## The gate

```ts
// workflows/refund.workflow.ts
import { z } from '@stackbone/sdk';
import { requestApproval } from '@stackbone/sdk/workflow';

export const inputSchema = z.object({ orderId: z.string(), amount: z.number().positive() });
export const outputSchema = z.object({ refunded: z.boolean(), decision: z.string() });

export async function refundWorkflow(input: z.infer<typeof inputSchema>) {
  'use workflow';

  const decision = await requestApproval({
    token: `refund-${input.orderId}`, // resume key — unique per approval in the run
    topic: 'refund',
    payload: { orderId: input.orderId, amount: input.amount },
    title: 'Approve refund',
    timeout: '24h', // ISO duration or ms
    fallback: 'reject', // applied if the timeout wins the race
  });

  // status === 'approved' is the ONLY green light; gate the side-effect on it.
  if (decision.status !== 'approved') {
    return { refunded: false, decision: decision.status };
  }
  await performRefund(input.orderId, input.amount); // a non-idempotent step, gated
  return { refunded: true, decision: decision.status };
}
```

## Options

| Option     | Required | Notes                                                                                              |
| ---------- | -------- | -------------------------------------------------------------------------------------------------- |
| `token`    | yes      | The resume key — **unique per approval within a run**. The host resumes the parked run by it.      |
| `topic`    | yes      | Approval category, mirrored to the inbox row (`'refund'`, `'contract.clause-edit'`).               |
| `payload`  | yes      | The data the human reviews in the inbox card. Keep it small and readable.                          |
| `title`    | no       | Human-readable title shown in the inbox / run inspection.                                          |
| `timeout`  | no       | ISO-8601 duration (`'24h'`, `'90m'`) or milliseconds. Omit for no deadline.                        |
| `fallback` | no       | `'approve'` \| `'reject'` — the decision applied if the timeout wins the race. Default `'reject'`. |

## The decision

`requestApproval` resolves to `{ status, payload?, timedOut }`:

- `status` is `'approved'` or `'rejected'`. **Only `'approved'` should unlock the side-effect.**
- `payload` is whatever the human attached when deciding (optional).
- `timedOut` is `true` only when the `fallback` was applied because nobody decided in time. A run that "resumed on its own and rejected" is the fallback firing — check the `timeout` / `fallback`.

## Resolving the decision

A human decides in the Studio HITL inbox, or from the shell:

```sh
stackbone hitl list --json                  # the pending inbox
stackbone hitl approve <hitlId> --yes        # ✱ resumes the parked run
stackbone hitl reject <hitlId> --reason "…" --yes
```

The host resumes the parked run by `token` (`POST /api/workflows/hooks/:token/resume`). See the **stackbone-cli** skill (`hitl` reference).

## Reading inbox state — `stackbone.approval`

`requestApproval` writes the inbox row through the ambient `stackbone.approval` surface; you **rarely call it directly**. When you need to read approval state from a tool or step, `stackbone.approval.get(approvalId)` / `stackbone.approval.list({ status, topic })` return the `{ data, error }` envelope. For a fully custom gate, the raw `defineHook` + `sleep` are also re-exported from `@stackbone/sdk/workflow`.

## Best practices

1. **Gate on `status === 'approved'`.** Treat rejected and timed-out the same way — no side-effect.
2. **Keep the gated step idempotent.** A resume replays the workflow body up to the gate; the side-effect must be safe to reach again.
3. **Use a stable, unique `token`** (e.g. `refund-${orderId}`) — it is the resume key and groups nothing else.
4. **Always set a `timeout` + `fallback`.** An approval with no deadline can park a run forever.
5. **Small, human-readable `payload`.** The approver scans a card — put rationale in `title`/`payload`, not a 200-line blob.

## Common mistakes

| Mistake                                            | Fix                                                                        |
| -------------------------------------------------- | -------------------------------------------------------------------------- |
| Calling `requestApproval` inside a `'use step'`    | It's a workflow primitive — call it from the `'use workflow'` body.        |
| Running the side-effect before checking `status`   | Gate it: `if (decision.status !== 'approved') return …` before the effect. |
| Reusing one `token` for several approvals in a run | Each approval needs its own unique `token` (it's the resume key).          |
| Expecting `timedOut` to mean "rejected by a human" | `timedOut: true` is the `fallback` firing, not a human decision.           |

## Common error codes

| `error.code`               | Cause                                                      | Action                                                                      |
| -------------------------- | ---------------------------------------------------------- | --------------------------------------------------------------------------- |
| `approval_persist_failed`  | The inbox row write failed (run never showed in the inbox) | A missing/unmigrated `approvals` table or DB error — inspect `error.cause`. |
| `approval_not_found`       | `stackbone.approval.get()` on an unknown id                | The id never persisted; treat as terminal.                                  |
| `approval_invalid_request` | Missing `topic` / `payload` on the request                 | Supply the required field.                                                  |
| `approval_unavailable`     | A read against the agent database failed                   | Retry; if it persists, the database handle is unavailable.                  |

See the main [SKILL.md](../SKILL.md) for how `requestApproval` fits the durable-workflow model, and [scheduling/sdk-integration.md](../scheduling/sdk-integration.md) for the other workflow helpers.
