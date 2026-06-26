# `stackbone.rag` — SDK integration

End-to-end document ingest + retrieval over the agent's own Neon. Composes:

- A built-in **parser** (`stackbone.rag.parse`) that turns text or a PDF into a clean string.
- A built-in **chunker** (`stackbone.rag.chunk`) that splits text into overlapping windows.
- **Embeddings** via `stackbone.ai.embeddings.create()` (OpenRouter), or precomputed vectors you supply.
- Storage in `pgvector` inside the agent's Neon, with an HNSW index for fast kNN.
- **Vector retrieval** — cosine kNN against the query embedding, in a single query.

`stackbone.rag` is the recommended path for any document Q&A flow. Hand-roll only when you need a custom schema (e.g. multi-tenant collections with per-row RLS).

`import { stackbone } from '@stackbone/sdk'`. Reach `stackbone.rag` from any tool's `execute()` or any workflow step. The RAG schema (the `rag_*` tables + the pgvector HNSW index) is platform-provisioned per install — there is no creator-side setup; the only knob is `rag.embeddingModel` in `agent.yaml`.

## Configuring the embedding model

The `rag:` block in `agent.yaml` is optional and defaults to `openai/text-embedding-3-small`:

```yaml
rag:
  embeddingModel: openai/text-embedding-3-small # default model for ingest + retrieve
```

If the negotiated protocol contract doesn't grant the `rag` capability, the SDK returns `error.code === 'capability_unavailable'` at runtime.

`stackbone.rag.parse()` handles `text/*` and PDF inputs (PDFs via the bundled `unpdf` extractor — text only, no OCR). For Office formats, scanned images, or anything else, pre-extract the text yourself and hand the string to `ingest()`.

## Schema

The RAG schema is **provisioned for you by the platform migrator on every install** — the `rag_chunks` table (one row per chunk: `doc_id`, `chunk_idx`, `namespace`, `content`, `embedding`, `metadata`), the `pgvector` HNSW index aligned with the cosine operator (`<=>`), and the async-ingest jobs table. There is **no creator-side schema step**.

Do not declare these tables by hand. The schema (column names, index opclass) is internal to `stackbone.rag` and changes between SDK versions. A missing table returns `rag_schema_missing` — that means the install's provisioning didn't run (a setup/platform issue), not something you fix with a migration command.

## Ingest

You hand `ingest()` the document `id` plus its chunks. Two shapes: pass string `chunks` + a `model` and the SDK embeds for you, or pass `chunks` with precomputed `embedding` arrays for full provider/dimension control. Re-ingesting with the same `id` replaces all that document's chunks atomically.

```ts
const text = await stackbone.rag.parse(file); // PDF/text → string
const pieces = stackbone.rag.chunk(text, { size: 512, overlap: 64 });

const { data, error } = await stackbone.rag.ingest({
  id: 'doc_q4-renewal',
  chunks: pieces,
  model: 'openai/text-embedding-3-small',
  metadata: { ownerEmail: 'alice@acme.com', deal: 'Q4-renewal' },
});

if (error) throw new Error(error.message);

// data === { id: 'doc_q4-renewal', chunks: 12 }
return { docId: data.id, chunks: data.chunks };
```

`parse()` and `chunk()` are pure helpers — `parse()` accepts a `Blob`, `string`, `Uint8Array` or `ArrayBuffer` (text passes through; PDFs are flattened to one string via `unpdf`; other MIME types throw), and `chunk()` splits into overlapping windows. `ingest()` runs the embed + persist round-trip and resolves once the chunks are written, so a document is retrievable the moment `ingest()` returns ok.

### Async ingest for large documents

Big documents should ingest in the background. `ingestAsync()` allocates a job row up front, returns immediately with a `jobId`, and exposes both a stream of progress events and a `result` promise:

```ts
const { data, error } = await stackbone.rag.ingestAsync({
  id: 'doc_big-pdf',
  collection: 'contracts', // required for ingestAsync — binds the job row
  chunks: pieces,
  model: 'openai/text-embedding-3-small',
  metadata: { name: 'big.pdf' },
});

if (error) throw new Error(error.message);

// data === { jobId, events: AsyncIterable<RagIngestProgress>, result: Promise<Result> }
return { jobId: data.jobId };
```

### Durable background ingest

From a workflow, call `ingestDocuments({ collection, content, contentType })` (or `{ collection, storageKey, contentType }`) from `@stackbone/sdk/workflow` — it stages the content to storage in a durable step and runs the reserved `rag-ingest` workflow:

```ts
import { ingestDocuments } from '@stackbone/sdk/workflow';

const { documentId, chunks } = await ingestDocuments({
  collection: 'contracts',
  content: pdfBytes,
  contentType: 'application/pdf',
});
```

### Progress and cancellation

The `events` channel yields `RagIngestProgress` records as the job advances: `{ type: 'started' | 'progress' | 'completed' | 'failed', jobId, ... }`. For UI progress, prefer subscribing in the frontend to a Studio Live channel; in the agent, consume `events` only when a downstream step truly needs to gate on completion (rare). A job is cancelled by flipping its row status from outside (`POST /api/rag/jobs/:jobId/cancel`); the pipeline observes it between chunk batches and ends with `rag_ingest_cancelled`.

### Accepted parser inputs

`stackbone.rag.parse()` accepts:

| Input                                         | Handling                                              |
| --------------------------------------------- | ----------------------------------------------------- |
| `string`                                      | Returned verbatim — fed straight into the chunker.    |
| `Blob`, `Uint8Array`, `ArrayBuffer` (text/\*) | Decoded as UTF-8 text.                                |
| `Blob`, `Uint8Array`, `ArrayBuffer` (PDF)     | Flattened to one string via `unpdf` (no OCR).         |
| Any other MIME type                           | Throws — pre-extract the text yourself before ingest. |

`metadata` is a free-form `Record<string, unknown>` stored per document, returned by `retrieve()`, and usable in `filter`.

## Retrieve

Pass the query `text` plus the same `model` you used at ingest, and `retrieve()` embeds it and runs a single vector kNN query, returning the top hits as an array:

```ts
const { data, error } = await stackbone.rag.retrieve({
  text: 'payment terms net 30',
  model: 'openai/text-embedding-3-small',
  topK: 5,
  filter: { deal: 'Q4-renewal' },
});

if (error) throw new Error(error.message);

for (const hit of data) {
  // hit === { id, chunkIdx, content, metadata, score }
}
```

Retrieval is a single Postgres query. It is **vector kNN** — the query embedding is matched against the chunk embeddings by cosine distance and ranked by similarity `score`. If you already have a query vector, pass `embedding` instead of `text` + `model` and the SDK skips the embed step.

### Options

| Option            | Default        | Notes                                                                                                                     |
| ----------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `topK`            | `5`            | Number of hits returned, ranked by similarity score.                                                                      |
| `filter`          | none           | Equality filter on `metadata`. `{ deal: 'Q4-renewal' }` becomes `metadata @> '{"deal":"Q4-renewal"}'`.                    |
| `namespace`       | `'default'`    | Logical separation — only chunks ingested under the same namespace are searched.                                          |
| `includeContent`  | `true`         | Pass `false` when you only need IDs to fetch the full document yourself.                                                  |
| `includeMetadata` | `true`         | Pass `false` to skip returning each hit's `metadata`.                                                                     |
| `model`           | matches config | Embedding model for the query. **Must equal the model used at ingest** — a dimension mismatch returns `rag_dim_mismatch`. |

### Distance operator alignment

The schema picks **cosine distance** (`<=>`) and builds the HNSW index with `vector_cosine_ops`. The retrieve query uses the same operator. You don't pick this — the SDK enforces it — but the alignment matters: querying with a different operator on a `vector_cosine_ops` index falls back to sequential scan and returns the wrong ranking.

## Delete

```ts
const { data, error } = await stackbone.rag.delete(docId);

if (error) throw new Error(error.message);
// data === { deleted: 1 }
```

Deletes the document's chunks and their embeddings (pass a single `docId` or an array). Idempotent — deleting a missing `docId` returns `{ deleted: 0 }` without error. To delete by metadata instead of id, use `stackbone.rag.deleteWhere(filter)`.

## Patterns

### Inject retrieved context into a chat call

```ts
const { data: hits, error } = await stackbone.rag.retrieve({
  text: question,
  model: 'openai/text-embedding-3-small',
  topK: 5,
});

if (error) throw new Error(error.message);

const context = hits.map((h) => `### ${h.id}\n${h.content}`).join('\n\n');

const { data: reply } = await stackbone.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages: [
    { role: 'system', content: `Answer using only the context below.\n\n${context}` },
    { role: 'user', content: question },
  ],
});

return { answer: reply.choices[0].message.content, citedDocIds: hits.map((h) => h.id) };
```

Cite the document ids back to the caller — RAG without citations breeds hallucinations.

### Multiple namespaces per agent

```ts
stackbone.rag.ingest({ id, chunks, model, namespace: 'contracts' }); // legal docs
stackbone.rag.ingest({ id, chunks, model, namespace: 'emails' }); // support threads
stackbone.rag.ingest({ id, chunks, model, namespace: 'internal-kb' }); // employee handbook
```

Pass `namespace` on both `ingest()` and `retrieve()` to keep separate corpora isolated within the same agent — only chunks ingested under a namespace are searched when you retrieve with it.

## Best practices

1. **Always destructure `{ data, error }`, then `throw new Error(error.message)` or branch.**
2. **Ingest big documents in the background** — `ingestAsync` (return the `jobId`), or the `ingestDocuments` workflow helper.
3. **Pin one embedding model per namespace.** Mixing dimensions corrupts the index.
4. **Always carry `metadata`.** Filters and citations need it; backfilling is painful.
5. **Cite document ids in LLM answers.** RAG only helps if the agent shows its sources.
6. **Use `stackbone.rag` instead of hand-rolling `pgvector`.** The SDK manages the operator + index + chunker alignment.
7. **Don't provision the schema yourself.** The RAG tables are platform-provisioned on every install; hand-written schema drifts from what `stackbone.rag` expects.

## Common mistakes

| Mistake                                                                                | Fix                                                                                      |
| -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Blocking a turn on a big inline `ingest()` round-trip                                  | Use `ingestAsync()` or the `ingestDocuments` workflow helper; let it finish out of band. |
| Embedding queries with `text-embedding-3-small` but indexing with `-3-large`           | Pick one model per namespace. `rag_dim_mismatch` if you mix.                             |
| Querying `rag_chunks` directly with `stackbone.database`                               | Use `stackbone.rag.retrieve(...)`. Internal schema, do not depend on it.                 |
| Passing an unsupported file type to `parse()`                                          | `parse()` handles text/\* and PDF; pre-extract the text yourself for anything else.      |
| Skipping `metadata` on ingest, then realising you cannot filter or attribute           | Always include `metadata` — at minimum `{ source, ownerId }`.                            |
| Hand-creating the `pgvector` HNSW index with `vector_ip_ops` while querying with `<=>` | Don't — the platform-provisioned schema pairs the operator + opclass for you.            |

## Common error codes

| `error.code`             | Cause                                                              | Action                                                                                                                               |
| ------------------------ | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `capability_unavailable` | The negotiated protocol contract doesn't grant `rag`               | Gating is via the contract handshake, not a manifest field — surface the message; if it persists it's a control-plane/install issue. |
| `rag_schema_missing`     | RAG tables not present on the install (provisioning didn't run)    | Platform-provisioned per install; a setup/platform issue, not a creator command.                                                     |
| `rag_invalid_request`    | Missing `id`, empty `chunks`, or a malformed retrieve query        | Fix the request shape per the message.                                                                                               |
| `rag_dim_mismatch`       | Query model dimensions ≠ stored index dimensions                   | Pin one embedding model per namespace; re-ingest if you change it.                                                                   |
| `rag_embedding_failed`   | The embeddings call returned the wrong number of vectors           | Inspect `error.meta` (expected vs received); retry the ingest.                                                                       |
| `rag_ingest_cancelled`   | An `ingestAsync` job's row was flipped to `cancelled` mid-run      | Expected when you cancel; restart the ingest if it was accidental.                                                                   |
| `rag_error`              | An uncaught Postgres / pipeline failure mapped to the generic code | Inspect `error.message`; check the schema is provisioned.                                                                            |

See the main [SKILL.md](../SKILL.md) for cross-module patterns. The chunker + embedding model pair is documented in [agent-yaml.md](../agent-yaml.md) under `rag:`.
