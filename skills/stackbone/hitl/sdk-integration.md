# `client.approval` — SDK integration

Human-in-the-loop pauses. This module is a **Stackbone-native facade** — it does not wrap any upstream SDK. The control plane persists the approval, surfaces it in the HITL inbox (Studio + email), and resumes the run when a human decides.

The key property: **pauses are durable**. The agent process can die, the container can hibernate, the org's tier can switch — the pending approval survives. When the human decides, the platform re-invokes the agent with the decision payload in `ctx.run.trigger === 'approval_resumed'`.

## Connection

```ts
import { createClient } from '@stackbone/sdk';

const client = createClient();
```

No env vars are needed. The platform routes approval traffic through `PLATFORM_API_URL` + `PLATFORM_API_KEY`, both injected at boot. You never touch them.

## Request and wait

The simplest shape — open an approval and block the current `invoke` until the human decides. Used when the invocation's deadline is generous (chat embeds, manual triggers) or when the work is small enough that holding the run is cheap.

```ts
const { data, error } = await client.approval.requestAndWait({
  title: 'Approve contract clause edit',
  description: 'Counterparty wants to remove the auto-renewal clause.',
  payload: { contractId, clauseDiff, requestedBy: 'alice@acme.com' },
  approverRole: 'approver',
});

if (error) return ctx.fail(error.code, error.message);

if (data.decision === 'approved') {
  await applyEdit(data.editedPayload ?? data.payload);
  return ctx.ok({ status: 'applied' });
}

if (data.decision === 'rejected') {
  return ctx.ok({ status: 'rejected', reason: data.reason });
}
```

`requestAndWait` returns when the approver decides. **The process holding the promise may not be the same process that resumes** — that is the durability contract. The SDK handles the continuation transparently:

1. `requestAndWait` registers the approval with the control plane and includes the current run's continuation token.
2. The platform persists `{ runId, continuationToken, approvalId }` in the HITL inbox.
3. If the container hibernates / crashes / scales down, the in-memory promise is lost — that is expected.
4. When the human decides, the platform re-invokes `/invoke` with `trigger === 'approval_resumed'` and the original input. The SDK detects the continuation token and the second call's `requestAndWait` resolves with the decision instead of opening a new approval.

You write the same code for both cases. The SDK abstracts the continuation.

### Options

| Option           | Required            | Notes                                                                                                           |
| ---------------- | ------------------- | --------------------------------------------------------------------------------------------------------------- |
| `title`          | yes                 | Shown in the inbox card. Keep ≤80 chars.                                                                        |
| `description`    | no                  | Markdown allowed. Goes under the title.                                                                         |
| `payload`        | yes                 | Any JSON-serialisable value. Shown to the approver verbatim; can be edited if `editable: true`.                 |
| `approverRole`   | yes                 | An RBAC role declared in the org (e.g. `'approver'`, `'finance-lead'`). Members with that role see the request. |
| `editable`       | no, default `false` | If true, the approver can edit `payload` before approving; the edited copy is in `data.editedPayload`.          |
| `expiresIn`      | no, default `7d`    | Auto-rejects after the deadline. String (`'48h'`) or `Date`.                                                    |
| `idempotencyKey` | no                  | Re-issuing `requestAndWait` with the same key resolves to the existing approval rather than opening a new one.  |

## Request (no wait)

Open the approval, return immediately, and continue the run. Useful when the agent has other work to do and should not block on the human.

```ts
const { data, error } = await client.approval.request({
  title: 'Verify identity document',
  payload: { userId, idDocKey },
  approverRole: 'kyc-reviewer',
});

if (error) return ctx.fail(error.code, error.message);

return ctx.ok({ approvalId: data.approvalId, status: 'pending' });
```

The platform calls `/invoke` again with `trigger === 'approval_resumed'` and the approval payload in `input.approval` when the decision lands. Handle that branch in your `invoke`:

```ts
async invoke(input, ctx) {
  if (ctx.run.trigger === 'approval_resumed') {
    const { approvalId, decision, payload, editedPayload } = input.approval;
    if (decision === 'approved') {
      await onApproved(editedPayload ?? payload);
    } else {
      await onRejected(payload, input.approval.reason);
    }
    return ctx.ok({});
  }

  // normal path…
}
```

## Resume programmatically

Useful for tests, for admin overrides, or for bot approvers (e.g. an auto-approver below a threshold).

```ts
const { data, error } = await client.approval.resume(approvalId, {
  decision: 'approved',
  reason: 'auto-approved: amount below threshold',
  editedPayload: { ...originalPayload, autoApproved: true },
});

if (error) return ctx.fail(error.code, error.message);
```

The platform treats programmatic `resume` exactly like a human decision — the original run resumes, the inbox card closes, the decision is auditable with `decidedBy: 'agent:<agentId>'`.

## Get

```ts
const { data, error } = await client.approval.get(approvalId);

if (error) return ctx.fail(error.code, error.message);

// data === { approvalId, status, payload, decision?, decidedBy?, decidedAt?, createdAt, expiresAt }
```

Useful when surfacing approval state to the UI from a non-resumed invocation.

## Cancel

```ts
const { data, error } = await client.approval.cancel(approvalId, 'user cancelled the request');

if (error) return ctx.fail(error.code, error.message);
```

Pulls the card from the inbox without inviting a decision. The original run is resumed with `decision === 'cancelled'`.

## Patterns

### Approve-or-die guard inside a longer flow

```ts
const { data, error } = await client.approval.requestAndWait({
  title: 'Send invoice for €10,000',
  payload: { invoiceId, amount: 10_000 },
  approverRole: 'finance-lead',
  expiresIn: '24h',
});

if (error) return ctx.fail(error.code, error.message);
if (data.decision !== 'approved') return ctx.ok({ status: 'skipped', reason: data.reason });

await sendInvoice(invoiceId);
return ctx.ok({ status: 'sent' });
```

The agent process can die between request and decision — the run resumes on the same code path 24 hours later if needed.

### Threshold-based auto-approval

```ts
async function approveOrAutoApprove(payload: { amount: number }, ctx) {
  if (payload.amount < 100) {
    // Cheap stuff doesn't need a human
    return { decision: 'approved' as const, payload };
  }

  const { data, error } = await client.approval.requestAndWait({
    title: `Charge €${payload.amount}`,
    payload,
    approverRole: 'finance-lead',
  });

  if (error) throw new Error(error.message);
  return { decision: data.decision, payload: data.editedPayload ?? data.payload };
}
```

### Idempotent approval (don't double-open)

```ts
const { data, error } = await client.approval.requestAndWait({
  title: 'Refund order',
  payload: { orderId },
  approverRole: 'support-lead',
  idempotencyKey: `refund:${orderId}`, // second call returns the existing approval
});
```

Without `idempotencyKey`, a retried `invoke` opens a duplicate card. With it, the second call rides the original approval.

## Best practices

1. **Always destructure `{ data, error }`.**
2. **Prefer `requestAndWait` for clarity.** The SDK handles continuation; you don't manage the two-call dance.
3. **Always set `approverRole`.** Without it, the request lands in a generic queue that no one watches.
4. **Use `idempotencyKey` whenever a retry could double-open.** Cheap insurance.
5. **Keep payloads small and human-readable.** The approver scans them in a card — JSON blobs of 200 lines are useless.
6. **Set an `expiresIn`.** Approvals that linger forever clog the inbox; 24 h–7 d is the right range for most flows.
7. **Surface decision context.** When you ask, include the `requestedBy` / `requestedAt` / business rationale in `payload` so the approver does not need to dig.

## Common mistakes

| Mistake                                                                       | Fix                                                                                                         |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Holding an in-memory queue of pending approvals                               | The platform persists them. Re-fetch with `client.approval.get(approvalId)` or wait for `approval_resumed`. |
| Polling `requestAndWait` in a loop                                            | `requestAndWait` is durable — call it once. The SDK handles continuation.                                   |
| Forgetting to handle `decision === 'rejected'`                                | The approver can reject. Always branch on `decision`.                                                       |
| Hard-coding `approverRole: 'admin'` everywhere                                | Different decisions need different reviewers (finance, KYC, legal). Pick the right role per request.        |
| Putting raw HTML / Markdown in `payload` and expecting it to render           | The card renders `payload` as JSON. Use `description` for rich text.                                        |
| Re-issuing the same approval on every `invoke` retry without `idempotencyKey` | Add an idempotency key — otherwise the inbox fills with duplicates.                                         |

## Common error codes

| `error.code`               | Cause                                                                                | Action                                                                     |
| -------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| `approval_already_decided` | `resume()` / `cancel()` on an approval that is already approved / rejected / expired | Inspect via `get()`; do not retry.                                         |
| `approval_not_found`       | `get()` / `resume()` / `cancel()` on an unknown ID                                   | The approval may have been deleted by retention policy. Treat as terminal. |
| `approval_expired`         | `requestAndWait` / `resume` after `expiresAt`                                        | Re-issue a new request if still relevant.                                  |
| `approval_role_unknown`    | `approverRole` does not exist in the org's RBAC                                      | Coordinate with the org member to declare the role in Studio.              |
| `capability_not_granted`   | `agent.yaml` missing `capabilities: [hitl]`                                          | Declare it — see [agent-yaml.md](../agent-yaml.md).                        |

See the main [SKILL.md](../SKILL.md) for the cross-module patterns. The HITL inbox itself is part of the platform, not the agent — orgs see all approvals across all installed agents in one place.
