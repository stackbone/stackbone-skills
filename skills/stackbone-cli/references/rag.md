# `stackbone rag`

Operate the managed Retrieval-Augmented Generation index bound to the targeted agent installation. Targets the local-dev install by default; override with `--agent <id>`. Document/job lists are paginated (`--limit` + `--cursor`). The destructive verbs — `collections remove`, `remove`, `retry`, `cancel` — require `--yes`.

> The RAG schema is platform-provisioned per install — you never migrate it (no `db add-rag`). This surface operates the **data** in that index, not its schema.

## Collections

| Command                                     | Description                                                                   |
| ------------------------------------------- | ----------------------------------------------------------------------------- |
| `stackbone rag collections list`            | Collections with per-collection doc + chunk counts.                           |
| `stackbone rag collections create <name>`   | Create an empty collection.                                                   |
| `stackbone rag collections remove <name>` ✱ | Delete a collection and every document under it (cascades). Requires `--yes`. |

## Documents, query, jobs

| Command                                           | Description                                                                                                                                                 |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `stackbone rag list --collection <c>`             | Documents in a collection. `--limit` (1-200), `--cursor`.                                                                                                   |
| `stackbone rag get <docId> --collection <c>`      | Download the original to `--out <path>` (else stdout) via a presigned URL.                                                                                  |
| `stackbone rag ingest <path> --collection <c>`    | Upload a local `.txt`/`.md`/`.pdf` (≤ 25 MiB), staging an async ingest job.                                                                                 |
| `stackbone rag query <text> --collection <c>`     | Similarity query — `<text>` is embedded server-side with your org key. `--topk` (1-50), `--model <id>`, or `--embedding <vector>` for a precomputed vector. |
| `stackbone rag remove <docId> --collection <c>` ✱ | Delete a document (cascades to its chunks). Requires `--yes`.                                                                                               |
| `stackbone rag jobs`                              | Async ingest jobs, optionally `--collection`. `--limit`, `--cursor`.                                                                                        |
| `stackbone rag retry <jobId>` ✱                   | Re-enqueue a failed ingest job from its stored original. Requires `--yes`.                                                                                  |
| `stackbone rag cancel <jobId>` ✱                  | Cancel a non-terminal ingest job. Requires `--yes`.                                                                                                         |

```sh
stackbone rag collections create docs --json
stackbone rag ingest ./handbook.pdf --collection docs --json   # { job_id, status }
stackbone rag jobs --collection docs --json
stackbone rag list --collection docs --json
```

`ingest` only accepts `.txt` / `.md` / `.pdf`; an unsupported extension fails CLI-side before any upload.

## Exit codes

| Code | When                                                                 |
| ---- | -------------------------------------------------------------------- |
| 0    | Success                                                              |
| 1    | Network / validation (missing `--collection`, unsupported file type) |
| 2    | Not authenticated                                                    |
| 3    | No install resolved — pass `--agent` or run `stackbone dev`          |
| 4    | Collection / document / job not found                                |
| 5    | A destructive verb without `--yes`                                   |
