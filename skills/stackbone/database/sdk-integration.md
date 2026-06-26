# `stackbone.database` — SDK integration

Drizzle ORM (driver `postgres-js`) over the agent's dedicated Neon Postgres. Exposes the relational store, vector store (`pgvector`), full-text store (`tsvector`) and KV cache (a table with `expires_at`) — all in the same database.

`stackbone.database` is **native Drizzle, verbatim**: awaiting a query returns the typed rows (an array) and **throws** on error. There is no `{ data, error }` envelope (that's every _other_ surface — `stackbone.storage`, `stackbone.ai`, `stackbone.rag`, `stackbone.approval`, `stackbone.secrets`, `stackbone.config`, `stackbone.prompts`). Handle errors with `try/catch`, or let the throw bubble up.

`import { stackbone } from '@stackbone/sdk'`. Reach `stackbone.database` from any tool's `execute()` or any workflow step — no `createClient()`, no connection string; the platform injects `DATABASE_URL` and the SDK builds + pools the connection lazily on first use (retrying Neon scale-to-zero wake-ups).

## Schema declaration

Declare your schema in `src/schema.ts` with Drizzle's typed builders, then **import the table objects** wherever you query. The verbs (`select().from(table)`, `insert(table)`, `update(table)`, `delete(table)`) take the `pgTable` **object** — never a string name.

```ts
import { pgTable, uuid, text, timestamp, jsonb, vector } from '@stackbone/sdk/db';

export const contracts = pgTable('contracts', {
  id: uuid('id').defaultRandom().primaryKey(),
  title: text('title').notNull(),
  body: text('body'),
  metadata: jsonb('metadata').$type<{ owner_email: string }>(),
  embedding: vector('embedding', { dimensions: 1536 }),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

`@stackbone/sdk/db` re-exports `drizzle-orm` and `drizzle-orm/pg-core` wholesale, so **every** column builder (`pgTable`, `text`, `uuid`, `vector`, …) and **every** query helper (`eq`, `and`, `or`, `sql`, `desc`, `asc`, `isNull`, …) come from that one path — never import `drizzle-orm` directly.

```ts
import { eq, and, desc, sql } from '@stackbone/sdk/db';
```

`stackbone db migrate create` reads this file to diff against the last applied migration, so edits here trigger a new migration on the next run.

## CRUD

Await the Drizzle builder directly — no `.first()` / `.all()` / `.page()`. A `select()` resolves to an **array** of typed rows; take the first with destructuring, check existence with `.length` or `if (!row)`.

```ts
import { eq, desc } from '@stackbone/sdk/db';

const rows = await stackbone.database.select().from(contracts).where(eq(contracts.id, contractId));
const [contract] = rows;
if (!contract) throw new Error(`Contract ${contractId} not found`);

// Pagination: Drizzle's own .limit() / .offset() / .orderBy() — there is no cursor helper.
const page = await stackbone.database
  .select()
  .from(contracts)
  .orderBy(desc(contracts.createdAt))
  .limit(50)
  .offset(0);
```

Wrap the await in `try/catch` only when you want to react to a DB error instead of letting it bubble.

`insert(table).values(...)` takes one object **or** an array; `.returning()` gives back the inserted rows:

```ts
const [created] = await stackbone.database
  .insert(contracts)
  .values({ title: 'NDA', body: '...' })
  .returning();

await stackbone.database.insert(contracts).values([{ title: 'A' }, { title: 'B' }]); // batch
```

`update` and `delete` follow the same shape — always pass an explicit `.where(...)`, add `.returning()` to get rows back:

```ts
import { eq, isNull } from '@stackbone/sdk/db';

const [updated] = await stackbone.database
  .update(contracts)
  .set({ status: 'signed', signedAt: new Date() })
  .where(eq(contracts.id, contractId))
  .returning();

await stackbone.database.delete(contracts).where(eq(contracts.id, contractId));
```

For soft-delete, add a `deleted_at` column and filter with `isNull(contracts.deletedAt)` — there's no built-in helper.

## Vectors (`pgvector`)

Declare the column with `vector('embedding', { dimensions: <N> })` where `<N>` matches your model:

| Model                           | Dimensions |
| ------------------------------- | ---------- |
| `openai/text-embedding-3-small` | 1536       |
| `openai/text-embedding-3-large` | 3072       |
| `cohere/embed-english-v3.0`     | 1024       |

Index in a migration with the **distance operator you'll query with** (`<->` cosine, `<#>` inner product, `<=>` L2), then query through `execute(sql\`...\`)`, which returns the result rows (and throws on error like every `stackbone.database` call):

```sql
CREATE INDEX contracts_embedding_idx ON contracts USING hnsw (embedding vector_cosine_ops);
```

```ts
import { sql } from '@stackbone/sdk/db';

const matches = await stackbone.database.execute(sql`
  SELECT id, title, embedding <-> ${queryEmbedding}::vector AS distance
  FROM contracts
  WHERE embedding IS NOT NULL
  ORDER BY embedding <-> ${queryEmbedding}::vector
  LIMIT 5
`);
```

> **An index built with one operator does sequential scans (and wrong results) for a query using a different one.** Pick the operator on day one and stay consistent. For RAG, prefer `stackbone.rag` — it aligns schema, index and operator for you.

## Full-text search (`tsvector`)

Drizzle's pg-core has no `tsvector` builder, so declare a `text` placeholder and let a migration trigger own the real column:

```ts
import { pgTable, uuid, text } from '@stackbone/sdk/db';

export const contracts = pgTable('contracts', {
  id: uuid('id').defaultRandom().primaryKey(),
  title: text('title').notNull(),
  body: text('body'),
  searchTsv: text('search_tsv'), // maintained by a trigger; Drizzle just needs to know it exists
});
```

```sql
ALTER TABLE contracts ALTER COLUMN search_tsv TYPE tsvector USING search_tsv::tsvector;

CREATE TRIGGER contracts_tsv_update BEFORE INSERT OR UPDATE
  ON contracts FOR EACH ROW EXECUTE FUNCTION
  tsvector_update_trigger(search_tsv, 'pg_catalog.english', title, body);
```

Query with the `@@` operator through raw SQL:

```ts
import { sql } from '@stackbone/sdk/db';

const results = await stackbone.database.execute(sql`
  SELECT id, title, ts_rank(search_tsv, plainto_tsquery('english', ${q})) AS rank
  FROM contracts
  WHERE search_tsv @@ plainto_tsquery('english', ${q})
  ORDER BY rank DESC
  LIMIT 10
`);
```

For **hybrid** search (union of vector kNN + tsvector top-K, re-ranked) reach for `stackbone.rag` — only hand-roll if you need a custom collection schema.

## KV cache (`expires_at` table)

For ad-hoc caching, declare your own table and read/write it with the normal verbs plus `onConflictDoUpdate` — no need for Redis, since the agent's Neon already exposes `pg_cron`:

```ts
import { eq, gt, and } from '@stackbone/sdk/db';

const cache = pgTable('app_cache', {
  key: text('key').primaryKey(),
  value: jsonb('value').notNull(),
  expiresAt: timestamp('expires_at').notNull(),
});

const [hit] = await stackbone.database
  .select()
  .from(cache)
  .where(and(eq(cache.key, key), gt(cache.expiresAt, new Date())));

await stackbone.database
  .insert(cache)
  .values({ key, value, expiresAt })
  .onConflictDoUpdate({ target: cache.key, set: { value, expiresAt } });
```

Add a `pg_cron` job in a migration to evict stale rows:

```sql
SELECT cron.schedule('cache-evict', '*/5 * * * *', $$ DELETE FROM app_cache WHERE expires_at < now() $$);
```

## Calling Postgres functions

There is no `.rpc(...)`. Call a stored function (defined in a migration) with raw SQL through `execute`:

```ts
import { sql } from '@stackbone/sdk/db';

const [{ total }] = await stackbone.database.execute(sql`
  SELECT compute_invoice_total(${contractId}) AS total
`);
```

## Transactions

Use a transaction when two or more writes must succeed or fail together. The callback gets `tx`, with the same verbs as `stackbone.database`; throwing inside rolls the whole thing back:

```ts
await stackbone.database.transaction(async (tx) => {
  const [contract] = await tx.insert(contracts).values({ title }).returning();
  await tx.insert(contractEvents).values({ contractId: contract.id, type: 'created' });
});
```

The SDK's auto-retry (Neon wake-up / pool rebuild) wraps the whole closure — don't add your own retry inside.

## Escape hatches

`stackbone.database.query` is Drizzle's relational query builder (`findFirst` / `findMany` over declared relations):

```ts
import { eq } from '@stackbone/sdk/db';

const contract = await stackbone.database.query.contracts.findFirst({
  where: eq(contracts.id, contractId),
});
```

`stackbone.database.raw()` and `.shared()` return the **raw Drizzle handle** (the same single pooled instance). Use `raw()` when a library needs a `PostgresJsDatabase` directly; `shared()` is the cross-surface accessor. Reach for them only when the structured verbs don't fit.

## Best practices

1. **`stackbone.database` throws — it does not return `{ data, error }`.** Let it bubble, or wrap in `try/catch`. The envelope is for every _other_ `stackbone.*` surface.
2. **Tables are objects, not strings.** Import your `pgTable` from `src/schema.ts` and pass the object to `select().from(table)`, etc.
3. **Helpers come from `@stackbone/sdk/db`.** `eq`, `and`, `sql`, `desc`, every column builder — one import path.
4. **A `select()` is an array.** Use `const [row] = await ...`; check `rows.length` / `if (!row)`. No `.first()` / `.all()` / `.page()`.
5. **Vector and full-text indexes are picky about operators.** If you build one, document the operator inline.
6. **Use `pg_cron` for cleanups.** It's already enabled — no external scheduler for db hygiene.
7. **Don't query internal Stackbone tables (`_storage_objects`, `_rag_*`, `_migrations`) by hand.** Use the `stackbone.storage` / `stackbone.rag` surfaces — those tables are internal contracts that can change.
8. **Migrations are the source of truth for schema.** Put every schema change through `stackbone db migrate create` — the read-only `stackbone db query` explorer can't alter schema, and changing it any other way leaves drift.

## Common mistakes

| Mistake                                                | Fix                                                                                                   |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| `const { data, error } = await stackbone.database...`  | It returns rows / throws — `const rows = await ...`, then `try/catch` if you need to handle the error |
| `.from(table).select()` / `.select().where().first()`  | `await stackbone.database.select().from(table).where(...)` → array; first row is `const [row] = rows` |
| `.all()` / `.page({ limit, cursor })`                  | `.limit(n).offset(m)` with `.orderBy(...)` — there is no cursor helper                                |
| `stackbone.database.rpc('fn', { ... })`                | `await stackbone.database.execute(sql\`SELECT fn(${arg})\`)`                                          |
| Passing a string table name (`.from('contracts')`)     | Pass the imported `pgTable` object (`.from(contracts)`)                                               |
| Index built `vector_l2_ops`, query uses `<->` (cosine) | Pick one and use it everywhere. Rebuild the index or rewrite the query.                               |
| Filter on `tsvector` column without `@@`               | Use `@@` with `plainto_tsquery` / `to_tsquery` in a raw `execute(sql\`...\`)`                         |
| `stackbone.database.delete()` without `.where(...)`    | Always pass an explicit `.where(...)`; an unfiltered delete/update wipes the whole table              |
| Reading `_rag_chunks` / internal tables directly       | Use `stackbone.rag` / `stackbone.storage` — the schema is internal                                    |
| Forgetting to `await` the builder                      | The Drizzle builder is a thenable; without `await` you hold the unresolved builder, not the rows      |

See the main [SKILL.md](../SKILL.md) for how `stackbone.database` sits alongside the other ambient surfaces.
