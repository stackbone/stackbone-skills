# `client.database` — SDK integration

Drizzle ORM (driver `postgres-js`) over the agent's dedicated Neon Postgres. Exposes the relational store, vector store (`pgvector`), full-text store (`tsvector`), KV cache (table with `expires_at`) and the durable queue table (`_queue_jobs`) — all in the same database.

## Connection

```ts
import { createClient } from '@stackbone/sdk';

const client = createClient(); // reads STACKBONE_POSTGRES_URL injected at boot
```

No connection string is passed. The platform injects `STACKBONE_POSTGRES_URL` and the SDK manages the pool, retries on Neon scale-to-zero wake-up (~1–3 s on cold), and HTTP-mode connection serialization for serverless wrappers.

## Schema declaration

Declare your schema in `src/db/schema.ts` using Drizzle's typed builders:

```ts
import { pgTable, uuid, text, timestamp, jsonb } from 'drizzle-orm/pg-core';
import { vector } from 'drizzle-orm/pg-core'; // pgvector

export const contracts = pgTable('contracts', {
  id: uuid('id').defaultRandom().primaryKey(),
  title: text('title').notNull(),
  body: text('body'),
  metadata: jsonb('metadata').$type<{ owner_email: string }>(),
  embedding: vector('embedding', { dimensions: 1536 }),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

The schema file is **read by `stackbone db migrate create`** to diff against the previous applied migration. Edits here trigger a new migration on the next `migrate create`.

## CRUD

All methods return `{ data, error }`. Always destructure both branches.

### Select

```ts
const { data, error } = await client.database
  .from(contracts)
  .select()
  .where(eq(contracts.id, contractId))
  .first();

if (error) return ctx.fail('db_read_failed', error.message);
if (!data) return ctx.fail('not_found', `Contract ${contractId}`);
return ctx.ok({ contract: data });
```

- `.first()` returns the first row or `null`.
- `.all()` returns the full array (use pagination for unbounded queries).
- `.page({ limit: 50, cursor })` returns `{ rows, nextCursor }` — cursor-based, opaque to the agent.

### Insert (always an array)

```ts
const { data, error } = await client.database
  .from(contracts)
  .insert([{ title: 'NDA', body: '...' }])
  .returning();

// `data` is `[{ id, title, body, ... }]` — note the array even for single inserts.
```

**Inserting a bare object is rejected**: `insert({ ... })` throws — the array shape is mandatory. This forces explicit batching semantics for fan-out and prevents accidental single-row inserts inside loops.

### Update

```ts
const { data, error } = await client.database
  .from(contracts)
  .update({ status: 'signed', signedAt: new Date() })
  .where(eq(contracts.id, contractId))
  .returning();
```

### Delete

```ts
await client.database.from(contracts).delete().where(eq(contracts.id, contractId));
```

For soft-delete patterns, add a `deleted_at` column and filter with `isNull(contracts.deletedAt)` — there is no built-in soft-delete helper.

## Vectors (`pgvector`)

Declare the column with `vector('embedding', { dimensions: <N> })` where `<N>` matches your embedding model:

| Model                           | Dimensions |
| ------------------------------- | ---------- |
| `openai/text-embedding-3-small` | 1536       |
| `openai/text-embedding-3-large` | 3072       |
| `cohere/embed-english-v3.0`     | 1024       |

Then index with the **distance operator you'll use at query time**:

```sql
-- In a migration, paired with the table:
CREATE INDEX contracts_embedding_idx
  ON contracts
  USING hnsw (embedding vector_cosine_ops);
```

Query with the matching operator (`<->` cosine, `<#>` inner product, `<=>` L2):

```ts
import { sql } from 'drizzle-orm';

const { data, error } = await client.database.execute(sql`
  SELECT id, title, embedding <-> ${queryEmbedding}::vector AS distance
  FROM contracts
  WHERE embedding IS NOT NULL
  ORDER BY embedding <-> ${queryEmbedding}::vector
  LIMIT 5
`);
```

> **An index built with one operator does sequential scans (and wrong results) for queries using a different operator.** Pick the operator on day one and stay consistent. For RAG, prefer `client.rag` — it manages the schema + index + operator alignment for you.

## Full-text search (`tsvector`)

```ts
import { tsvector } from '@stackbone/sdk/drizzle'; // re-exported helper

export const contracts = pgTable('contracts', {
  // ...
  searchTsv: tsvector('search_tsv'),
});

// In the migration, add a trigger:
// CREATE TRIGGER contracts_tsv_update BEFORE INSERT OR UPDATE
//   ON contracts FOR EACH ROW EXECUTE FUNCTION
//   tsvector_update_trigger(search_tsv, 'pg_catalog.english', title, body);
```

Query with the `@@` operator:

```ts
const { data, error } = await client.database.execute(sql`
  SELECT id, title, ts_rank(search_tsv, plainto_tsquery('english', ${q})) AS rank
  FROM contracts
  WHERE search_tsv @@ plainto_tsquery('english', ${q})
  ORDER BY rank DESC
  LIMIT 10
`);
```

## Hybrid search (vector + full-text)

The standard pattern: take the union of vector kNN and tsvector top-K, then re-rank. Built into `client.rag` already — only hand-roll if you need a custom collection schema.

## KV cache (`expires_at` table)

For ad-hoc caching outside the SDK's RAG / queue tables, declare your own:

```ts
export const cache = pgTable('app_cache', {
  key: text('key').primaryKey(),
  value: jsonb('value').notNull(),
  expiresAt: timestamp('expires_at').notNull(),
});
```

And add a `pg_cron` job in a migration to evict stale rows:

```sql
SELECT cron.schedule(
  'cache-evict',
  '*/5 * * * *',
  $$ DELETE FROM app_cache WHERE expires_at < now() $$
);
```

This is the recommended pattern instead of pulling in Redis — the agent's Neon already exposes `pg_cron`.

## RPC (Postgres functions / triggers)

```ts
const { data, error } = await client.database.rpc('compute_invoice_total', {
  contract_id: contractId,
});
```

Functions live in a migration:

```sql
CREATE OR REPLACE FUNCTION compute_invoice_total(contract_id uuid)
  RETURNS numeric
  LANGUAGE sql
  STABLE
AS $$
  SELECT coalesce(sum(amount), 0) FROM invoice_lines WHERE contract_id = $1
$$;
```

## Transactions

```ts
await client.database.transaction(async (tx) => {
  await tx.insert(contracts).values([{ title }]);
  await tx.insert(contractEvents).values([{ contractId, type: 'created' }]);
});
```

Rolls back on any thrown error. The SDK's auto-retry (Neon hibernation) wraps the whole closure — don't add your own retry inside.

## Best practices

1. **Always destructure `{ data, error }`.** Throw or return on error; never proceed with `data` when `error` is set.
2. **Inserts are arrays.** Single inserts use `.insert([{...}])` — the array shape is enforced.
3. **Vector and full-text indexes are picky about operators.** If you build one, document the operator inline.
4. **Use `pg_cron` for cleanups.** It's already enabled. No need for an external scheduler for db hygiene.
5. **Don't query Stackbone tables (`_storage_objects`, `_rag_*`, `_queue_jobs`, `_migrations`) by hand.** Use the `client.storage` / `client.rag` / `client.queues` modules. Those tables are internal contracts that can change.
6. **Migrations are the source of truth for schema.** Don't apply ad-hoc DDL via `stackbone db query --writable` — it works but leaves drift.

## Common mistakes

| Mistake                                                | Fix                                                                                                                                |
| ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| `insert({ title })` (object)                           | `insert([{ title }])` (array)                                                                                                      |
| Index built `vector_l2_ops`, query uses `<->` (cosine) | Pick one and use it everywhere. Rebuild the index or rewrite the query.                                                            |
| Filter on `tsvector` column without `@@`               | Use `@@` with `plainto_tsquery` / `to_tsquery`                                                                                     |
| `client.database.delete()` without `.where(...)`       | The SDK refuses unfiltered `delete()` / `update()` calls — pass an explicit `.where(eq(true, true))` if you really mean "all rows" |
| Reading `_queue_jobs` / `_rag_chunks` directly         | Use `client.queues` / `client.rag` — schema is internal                                                                            |
| Forgetting to `await` on a chain                       | All chain terminators (`.first()`, `.all()`, `.page()`, `.returning()`) are async; missing `await` returns the unresolved builder  |
