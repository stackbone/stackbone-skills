# `client.rag` — SDK integration

End-to-end document ingest + retrieval over the agent's own Neon. Composes:

- **LlamaParse** (opt-in) for PDFs / Office / scanned images — turns a binary into clean text + tables.
- A built-in **chunker** that splits text into overlapping windows.
- **Embeddings** via `client.ai.embeddings.create()` (OpenRouter).
- Storage in `pgvector` (vector kNN) + `tsvector` (full-text) inside the agent's Neon.
- **Hybrid retrieval** — vector + lexical with reciprocal rank fusion, in a single query.

`client.rag` is the recommended path for any document Q&A flow. Hand-roll only when you need a custom schema (e.g. multi-tenant collections with per-row RLS).

## Opting into the parser

Declare in `agent.yaml`:

```yaml
capabilities:
  - rag
rag:
  parser: llamaparse # or `none`
```

- **`llamaparse`** — the platform injects `LLAMA_PARSE_API_KEY`. `upload()` runs through LlamaParse first, so PDFs/Office/images become structured text with tables preserved. Counts against the org's parse quota.
- **`none`** — no parser key is injected. `upload()` accepts only `text/*` and `application/json` content. Cheaper, but you provide the text yourself.

If you call `client.rag.from(...)` without declaring `capabilities: [rag]`, the SDK returns `error.code === 'capability_not_granted'` at runtime.

## Schema

`stackbone db add-rag --name <collection>` generates the correct schema for you — three tables (`<collection>_documents`, `<collection>_chunks`, `<collection>_ingest_jobs`) plus the right `pgvector` HNSW index aligned with the cosine operator (`<->`). Re-run the command for each new collection. The migration is staged like any other Drizzle migration; `stackbone db migrate apply` ships it.

Do not declare these tables by hand. The schema (column names, index opclass, FK shape) is internal to `client.rag` and changes between SDK versions.

## Ingest (async)

```ts
const { data, error } = await client.rag.from('contracts').upload(file, {
  metadata: { ownerEmail: 'alice@acme.com', deal: 'Q4-renewal' },
});

if (error) return ctx.fail(error.code, error.message);

// data === { docId: 'doc_…', status: 'queued', jobId: 'job_…' }
return ctx.ok({ docId: data.docId });
```

`upload()` is **async** — it enqueues parsing + chunking + embedding via QStash and returns immediately with a `docId`. The document becomes retrievable once the job lands; until then, it does not appear in `retrieve()` results.

Never block `invoke` waiting for an ingest to finish. The wrapper deadline (~30 s) is much shorter than a 200-page PDF parse (minutes).

### Polling ingest status

```ts
const { data, error } = await client.rag.from('contracts').jobStatus(jobId);

if (error) return ctx.fail(error.code, error.message);

// data === { jobId, status: 'queued' | 'parsing' | 'embedding' | 'done' | 'failed', error? }
```

For UI progress, prefer subscribing in the frontend to a Studio Live channel; in the agent, poll only when a downstream step truly needs to gate on ingest completion (rare).

### Accepted inputs

| Input                                                                                        | Parser required?                                         |
| -------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `File`, `Blob`, `Uint8Array`, `Buffer`, `Readable`                                           | Depends on `contentType`                                 |
| `{ contentType: 'text/plain', body: '...' }`                                                 | No — fed straight into the chunker                       |
| `{ contentType: 'application/pdf', body: <bytes> }`                                          | **Yes** (`rag.parser: llamaparse`)                       |
| `{ contentType: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document' }` | **Yes** (LlamaParse handles `.docx` / `.pptx` / `.xlsx`) |
| `{ contentType: 'image/png' }`                                                               | **Yes** (LlamaParse OCR)                                 |

`metadata` is a free-form `Record<string, unknown>` returned by `retrieve()` and usable in `filter`.

## Retrieve (sync)

```ts
const { data, error } = await client.rag.from('contracts').retrieve('payment terms net 30', {
  limit: 5,
  filter: { deal: 'Q4-renewal' },
});

if (error) return ctx.fail(error.code, error.message);

for (const hit of data) {
  // hit === { docId, chunkIdx, content, metadata, score }
}
```

Retrieval is fast (single Postgres query, hybrid scored). Hybrid means: vector kNN against the query embedding + `ts_rank` against the same query as a `tsquery`, fused with reciprocal rank fusion. You get semantic and lexical matches in one call without writing the SQL.

### Options

| Option           | Default        | Notes                                                                                                                                          |
| ---------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `limit`          | `5`            | Top-K returned hits across the fused ranking.                                                                                                  |
| `filter`         | none           | Equality filter on `metadata`. `{ deal: 'Q4-renewal' }` becomes `metadata @> '{"deal":"Q4-renewal"}'`.                                         |
| `mode`           | `'hybrid'`     | `'hybrid'` \| `'vector'` \| `'lexical'`. Drop to `'vector'` only when the query is structurally non-lexical (e.g. similarity to an embedding). |
| `includeContent` | `true`         | Pass `false` when you only need IDs to fetch the full document yourself.                                                                       |
| `model`          | matches ingest | Embedding model for the query. **Must equal the model used at ingest** — dimension mismatch returns `rag_dimension_mismatch`.                  |

### Distance operator alignment

The schema generator picks **cosine distance** (`<->`) and builds the HNSW index with `vector_cosine_ops`. The retrieve query uses the same operator. You don't pick this — the SDK enforces it — but the alignment matters: querying with `<#>` (inner product) on a `vector_cosine_ops` index falls back to sequential scan and returns the wrong ranking.

If you need a different distance, regenerate the collection with `stackbone db add-rag --name <col> --distance ip|l2` and re-ingest.

## Delete

```ts
const { data, error } = await client.rag.from('contracts').delete(docId);

if (error) return ctx.fail(error.code, error.message);
// data === { deleted: 1 }
```

Deletes the document, its chunks, and its embeddings. Idempotent — deleting a missing `docId` returns `{ deleted: 0 }` without error.

## Patterns

### Inject retrieved context into a chat call

```ts
const { data: hits, error } = await client.rag
  .from('contracts')
  .retrieve(userQuestion, { limit: 5 });

if (error) return ctx.fail(error.code, error.message);

const context = hits.map((h) => `### ${h.docId}\n${h.content}`).join('\n\n');

const { data: reply } = await client.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages: [
    { role: 'system', content: `Answer using only the context below.\n\n${context}` },
    { role: 'user', content: userQuestion },
  ],
});

return ctx.ok({ answer: reply.choices[0].message.content, citedDocIds: hits.map((h) => h.docId) });
```

Cite the `docId`s back to the caller — RAG without citations breeds hallucinations.

### Ingest from a queue receiver

For batch ingest (e.g. "sync 1000 docs from GDrive"), publish one QStash job per document and let the receiver call `upload()`:

```ts
// In invoke: fan out
for (const file of files) {
  await client.queues.publish('ingest-doc', { url: file.url, name: file.name });
}

// In receivers
export const receivers = {
  'ingest-doc': async (payload, ctx) => {
    const file = await fetch(payload.url).then((r) => r.blob());
    const { error } = await client.rag
      .from('contracts')
      .upload(file, { metadata: { name: payload.name } });
    if (error) ctx.logger.warn('ingest failed', { name: payload.name, code: error.code });
  },
};
```

Each ingest is a separate QStash job — failures retry independently, the wrapper deadline applies per document.

### Multiple collections per agent

```ts
client.rag.from('contracts'); // legal docs
client.rag.from('emails'); // customer support thread store
client.rag.from('internal-kb'); // employee handbook
```

Each is a separate schema (own tables, own index). Run `stackbone db add-rag --name <each>` once at scaffolding time.

## Best practices

1. **Always destructure `{ data, error }`.**
2. **Ingest is async; retrieve is sync.** Never block `invoke` on ingest completion.
3. **Pin one embedding model per collection.** Mixing dimensions corrupts the index.
4. **Always carry `metadata`.** Filters and citations need it; backfilling is painful.
5. **Cite `docId`s in LLM answers.** Hybrid RAG only helps if the agent shows its sources.
6. **Use `client.rag` instead of hand-rolling `pgvector`.** The SDK manages the operator + index + chunker alignment.
7. **Provision collections via `stackbone db add-rag`.** Hand-written schema drifts from what `client.rag` expects.

## Common mistakes

| Mistake                                                                                | Fix                                                                                |
| -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Calling `upload()` and awaiting before responding to the user                          | `upload()` is async; return `docId` and let the job complete in the background.    |
| Embedding queries with `text-embedding-3-small` but indexing with `-3-large`           | Pick one model per collection. `rag_dimension_mismatch` if you mix.                |
| Querying `_rag_chunks` directly with `client.database`                                 | Use `client.rag.from(...).retrieve(...)`. Internal schema, do not depend on it.    |
| Uploading a PDF with `rag.parser: none`                                                | Either set `rag.parser: llamaparse` or pre-extract text yourself before upload.    |
| Skipping `metadata` on ingest, then realising you cannot filter or attribute           | Always include `metadata` — at minimum `{ source, ownerId }`.                      |
| Hand-creating the `pgvector` HNSW index with `vector_ip_ops` while querying with `<->` | Let `stackbone db add-rag` generate the schema; the operator + opclass are paired. |

## Common error codes

| `error.code`               | Cause                                                          | Action                                                                                 |
| -------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `capability_not_granted`   | `agent.yaml` missing `capabilities: [rag]`                     | Declare it — see [agent-yaml.md](../agent-yaml.md).                                    |
| `rag_parser_not_enabled`   | `upload()` got a PDF/image but `rag.parser: none`              | Set `rag.parser: llamaparse` and republish, or pre-extract text.                       |
| `rag_collection_not_found` | `from('xyz')` before running `stackbone db add-rag --name xyz` | Generate the schema and apply the migration.                                           |
| `rag_dimension_mismatch`   | Query model dimensions ≠ index dimensions                      | Pin one embedding model per collection; re-ingest if you change it.                    |
| `rag_ingest_failed`        | LlamaParse or embedding upstream errored                       | Inspect `jobStatus(jobId)`; retry with `client.rag.from(...).upload(...)` on a new ID. |
| `rag_quota_exceeded`       | Org tier parse quota hit                                       | Surface verbatim; the org member must upgrade.                                         |

See the main [SKILL.md](../SKILL.md) for cross-module patterns. The chunker + embedding model pair is documented in [agent-yaml.md](../agent-yaml.md) under `rag:`.
