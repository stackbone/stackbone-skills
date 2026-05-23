# `client.queues` — SDK integration

`@upstash/qstash` (publisher) + a Hono receiver with HMAC verification via Web Crypto. The canonical primitive for **work that has to outlive a single invocation**: anything > 30 seconds, cross-container fan-out, scheduled callbacks, or retries with backoff.

The job's source of truth is the `_queue_jobs` table inside the agent's Neon — QStash is the **push dispatcher** that wakes the hibernated container with a signed HTTP callback. `client.queues` hides both pieces behind one API.

## Why QStash and not `pg_cron` for callbacks

Neon scales the agent's Postgres to zero when idle. `pg_cron` fires inside Postgres — it cannot wake the container if the container is hibernating. QStash sends an authenticated HTTP request to `/invoke`, which wakes the container. **Use QStash for any callback that must invoke the agent**; use `pg_cron` only for in-database housekeeping (e.g. evicting cache rows).

## Connection

```ts
import { createClient } from '@stackbone/sdk';

const client = createClient();
```

The platform injects three env vars at boot:

| Env var                      | Role                                            |
| ---------------------------- | ----------------------------------------------- |
| `QSTASH_TOKEN`               | Publisher token (sign + submit jobs)            |
| `QSTASH_CURRENT_SIGNING_KEY` | Active HMAC key for the receiver                |
| `QSTASH_NEXT_SIGNING_KEY`    | Rotation slot — accept both during key rotation |

You never read these directly. `client.queues.publish(...)` reads `QSTASH_TOKEN`; the receiver shipped by `defineAgent` verifies signatures against both keys.

## Publish

```ts
const { data, error } = await client.queues.publish('process-contract', { contractId, userId });

if (error) return ctx.fail('queue_publish_failed', error.message);

// data === { messageId: 'msg_…', scheduledFor?: Date }
return ctx.ok({ jobId: data.messageId });
```

The first argument is the **topic** — a string that maps to a key in the agent's `receivers` export (see below). The second is the payload (any JSON-serialisable value).

### Delay and scheduling

```ts
// Run in 5 minutes
await client.queues.publish('send-followup', { leadId }, { delay: '5m' });

// Run at a specific time
await client.queues.publish(
  'send-reminder',
  { invoiceId },
  { schedule: new Date('2026-06-01T09:00:00Z') },
);
```

| Option            | Type                                                                  | Notes                                                                                                   |
| ----------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `delay`           | `string` (duration: `'30s'`, `'5m'`, `'2h'`, `'1d'`) or `number` (ms) | Relative — fired `delay` from publish time.                                                             |
| `schedule`        | `Date`                                                                | Absolute — fired at the given timestamp. Mutually exclusive with `delay`.                               |
| `retries`         | `number` (default 3)                                                  | Per-job override of the org's retry policy.                                                             |
| `deduplicationId` | `string`                                                              | QStash drops duplicates within 7 days. Use `${topic}:${entityId}` to enforce idempotency at the source. |

### Recurring schedules

```ts
await client.queues.schedule('daily-reconciliation', '0 9 * * *', async (ctx) => {
  // Runs every day at 09:00 UTC
  await reconcile(ctx);
});
```

The handler closure is **declared** at agent boot, not registered dynamically per invocation. Stackbone reads `schedules` (declared via `client.queues.schedule()` at module-load time inside `defineAgent`) and provisions QStash cron triggers at deploy time. Editing the schedule string requires a republish.

Prefer `client.queues.schedule` over `pg_cron` whenever the cron tick must wake the container — `pg_cron` cannot.

## Receive

The receiver lives **inside the agent**, exported next to `defineAgent`. The wrapper routes incoming QStash callbacks to it, verifies the HMAC signature, parses the payload, and calls the right handler.

```ts
import { defineAgent } from '@stackbone/sdk';

export default defineAgent({
  invoke: {
    /* ... */
  },
});

// Topic → handler. One handler per topic. Strongly typed against the payload.
export const receivers = {
  'process-contract': async (payload: { contractId: string; userId: string }, ctx) => {
    const { error } = await processContract(payload.contractId, ctx);
    if (error) {
      ctx.logger.warn('process-contract failed', { code: error.code });
      throw new Error(error.message); // throwing requests a retry
    }
    ctx.logger.info('process-contract done');
  },

  'send-followup': async (payload: { leadId: string }, ctx) => {
    await sendFollowup(payload.leadId, ctx);
  },
};
```

### Retries and the dead-letter loop

Throwing inside a receiver tells QStash to retry with exponential backoff (defaults: 3 attempts, 30 s / 5 m / 1 h). After the final failure, the job lands in the dead-letter view inside Studio — the org member can replay or discard it.

Return normally (no throw) to acknowledge the job. Returning a `Promise.resolve(undefined)` is the success signal.

### Idempotency

The wrapper passes the same payload on retry. **Make handlers idempotent.** Recommended pattern: dedupe at the database level with a unique index on `(topic, key)`:

```ts
'send-email': async (payload, ctx) => {
  const { error } = await client.database
    .from(emailLog)
    .insert([{ topic: 'send-email', key: payload.idempotencyKey, sentAt: new Date() }])
    .returning();

  if (error?.code === '23505') return; // unique violation — already sent
  if (error) throw new Error(error.message);

  await sendEmail(payload);
},
```

## Patterns

### Fan out work in `invoke`, do it in receivers

```ts
// invoke responds in <30 s; the work happens in the background
async invoke(input, ctx) {
  for (const item of input.items) {
    const { error } = await client.queues.publish(
      'process-item',
      { itemId: item.id },
      { deduplicationId: `process-item:${item.id}` },
    );
    if (error) ctx.logger.warn('publish failed', { itemId: item.id, code: error.code });
  }
  return ctx.ok({ enqueued: input.items.length });
}

export const receivers = {
  'process-item': async (payload, ctx) => {
    // Heavy lifting here — has its own per-job deadline
    await processItem(payload.itemId, ctx);
  },
};
```

This is the canonical shape for any operation that exceeds the invocation budget: ack fast, do work via queues.

### Cross-container event handoff

Two agents in the same org can hand work via QStash, but **`client.events.emit()` is the right primitive** for cross-agent fan-out — it goes through the org's typed event bus. Use `client.queues` for work the **same** agent owns (its own background jobs, its own retries).

### Scheduled callback that wakes the container

```ts
await client.queues.schedule('hourly-sync', '0 * * * *', async (ctx) => {
  const { error } = await syncFromGdrive(ctx);
  if (error) throw new Error(error.message); // QStash retries with backoff
});
```

Cron expressions follow standard 5-field syntax. Timezone is UTC.

## Best practices

1. **Always destructure `{ data, error }`.** Publish failures matter — losing a job silently is worse than retrying.
2. **Use `deduplicationId` on every publish.** It is QStash's only built-in idempotency hook; pair it with handler-side dedup for full safety.
3. **Make receivers idempotent.** Retries will replay the same payload; design for double-execution.
4. **Throw to retry, return to ack.** No other contract.
5. **Keep payloads small (<256 KB).** Pass IDs, not blobs — fetch the heavy data from `client.database` or `client.storage` inside the handler.
6. **Prefer `client.queues.schedule` over `pg_cron` for any cron that must invoke the agent.** `pg_cron` cannot wake hibernated containers.
7. **Don't poll `_queue_jobs` with `client.database`.** Internal schema; use the Studio queues inspector or `stackbone queues ls`.

## Common mistakes

| Mistake                                                   | Fix                                                                                                                         |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Returning a 200 from a handler that actually failed       | Throw — silent acks lose data.                                                                                              |
| Storing a 5 MB PDF in the payload                         | Upload to `client.storage`, queue the key.                                                                                  |
| Adding a `setTimeout(handler, 5*60*1000)` for delays      | Use `client.queues.publish(topic, payload, { delay: '5m' })`. The wrapper kills the invocation long before the timer fires. |
| Mutating shared state in a handler without idempotency    | Add a unique key (`deduplicationId` + DB unique index) before mutating.                                                     |
| Using `pg_cron` to call `/invoke` via HTTP                | Cron lives in Postgres; the container may be hibernating. Use `client.queues.schedule()`.                                   |
| Registering a receiver for a topic that nothing publishes | Dead code — publish a test job from `invoke` to confirm wiring.                                                             |

## Common error codes

| `error.code`              | Cause                                         | Action                                                                                                                              |
| ------------------------- | --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `queue_publish_failed`    | QStash REST API rejected the publish          | Inspect `error.details`; retry if `error.retryable === true`.                                                                       |
| `queue_signature_invalid` | Receiver got a callback with a bad HMAC       | The wrapper rejects before your handler runs. If you see it logged repeatedly, rotate `QSTASH_*_SIGNING_KEY` via the control plane. |
| `queue_topic_unknown`     | Publish to a topic with no matching receiver  | Add a handler in `receivers`, or fix the topic string.                                                                              |
| `queue_quota_exceeded`    | Org's monthly QStash budget hit               | Surface verbatim; the org member must upgrade.                                                                                      |
| `queue_payload_too_large` | Payload > 1 MB (QStash limit)                 | Upload to `client.storage`, queue the key.                                                                                          |
| `capability_not_granted`  | `agent.yaml` missing `capabilities: [queues]` | Declare it — see [agent-yaml.md](../agent-yaml.md).                                                                                 |

See the main [SKILL.md](../SKILL.md) for cross-module patterns. For the difference between QStash work queues and the org's event bus, see `client.events` in the SKILL's module reference.
