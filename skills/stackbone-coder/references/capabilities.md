# Capability checklist — walk it one at a time

After the type-specific interview, go through **every** capability below with the user, **one question each**: _do we need this?_ For each **yes**, note the surface, where it's reachable from, and what to add — then write the code following the named **stackbone**-skill deep-dive (don't reinvent the API; that doc is the source of truth). Default everything to **no** — only wire what the user confirms.

Two facts that decide most of this:

- **Where am I calling from?** A tool's `execute()` and a workflow's `'use step'` can use almost every surface. A handful are **workflow-body only**.
- **The envelope rule.** Every surface returns `{ data, error }` — destructure and handle both branches — **except** `stackbone.database`, which is native Drizzle (returns rows, **throws** on error).

All of these are reached through the ambient client: `import { stackbone } from '@stackbone/sdk'`. Never wire credentials or connection strings — the runtime injects them.

---

## A. Available from a tool **or** a workflow step

Ask each as "Do we need to …?"

| #   | Ask                                                               | Surface                                                                                                                   | Adds                                                                                                                              | Deep-dive (`stackbone` skill)                    |
| --- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| 1   | **store / query data?** (records, vectors, full-text)             | `stackbone.database` — native Drizzle over Neon Postgres (`pgvector` + `tsvector`)                                        | a Drizzle schema at `agents/<name>/schema.ts` (`pgTable` objects); migrations run on `stackbone dev`                              | `database/sdk-integration.md`                    |
| 2   | **upload / serve files?** (images, PDFs, exports)                 | `stackbone.storage` — `.from(bucket).upload/download/signed URL` (R2 in prod, MinIO in dev)                               | nothing to declare; bucket is a logical key prefix                                                                                | `storage/sdk-integration.md`                     |
| 3   | **make a one-off LLM call, embeddings, or vision?**               | `stackbone.ai` — chat / embeddings / vision via OpenRouter (300+ models), streamable                                      | nothing; uses the managed `OPENROUTER_API_KEY`                                                                                    | `ai/sdk-integration.md`                          |
| 4   | **search ingested documents?** (RAG)                              | `stackbone.rag` — ingest + hybrid (`pgvector` + `tsvector`) retrieval                                                     | `rag.embeddingModel` in `agents/<name>/agent.yaml`; ingest via the durable `ingestDocuments(...)` trigger                         | `rag/sdk-integration.md`                         |
| 5   | **call a third-party API?** (Slack, email, a CRM)                 | `stackbone.connection(id).<op>(args)` — brokered by Stackbone Connect; **throws** `ConnectorCallError` (match `err.code`) | the connector listed as required in `agents/<name>/agent.yaml`; types generated into `.stackbone/connect.d.ts` on `stackbone dev` | `connections/sdk-integration.md`                 |
| 6   | **use a versioned, editable prompt catalog?**                     | `stackbone.prompts` — `get()` / `compile(key, vars)` / `list()` / `create()`                                              | nothing to declare                                                                                                                | `prompts/sdk-integration.md`                     |
| 7   | **read operator-tunable config?** (greeting, limits, toggles)     | `stackbone.config` — `get(key)` / `getAll()`, key-checked + typed                                                         | a Zod object at `config.schema.ts` (workspace root); `stackbone dev` turns it into the Studio editor + typed `ConfigRegistry`     | `SKILL.md` → "Typed config — `config.schema.ts`" |
| 8   | **read operator-managed secrets?** (an API key the operator sets) | `stackbone.secrets` — `get(name)` / `getMany(names)`                                                                      | nothing in code; the operator sets the secret value in Studio                                                                     | `SKILL.md` → ambient surfaces table              |
| 9   | **call another agent?** (delegate a sub-task)                     | `stackbone.agent(id).session().send({ message, outputSchema })` then `result()`                                           | the sibling agent must exist in the workspace; from a tool **or** a step                                                          | `workflow-agents/authoring.md`                   |

> Note on #3 vs. the agent's own model: an **agent** already has a model bound in `agent.ts` for its conversation. Reach for `stackbone.ai` only when you need a _separate_ LLM call inside a tool/step, or embeddings/vision.

## B. Workflow-body **only** (not available in an agent tool)

| #   | Ask                                                        | Surface                                                                                                  | Notes                                                                                                                                            | Deep-dive                       |
| --- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------- |
| 10  | **pause for a human to approve / decide?** (HITL)          | `requestApproval({ token, topic, payload, timeout, fallback })` from **`@stackbone/sdk/workflow`**       | call from the workflow **body**, never inside a `'use step'`; needs the `workflow` peer installed; races the human decision against a timeout    | `hitl/sdk-integration.md`       |
| 11  | **kick off or schedule other workflows?** (fan-out / cron) | `stackbone.workflows.start(name, input)` · `.startAndWait(name, input)` · `.schedule(name, input, cron)` | from the workflow body (binds on dispatch); for ship-with schedules prefer the declarative `export const schedules = [...]` next to the workflow | `scheduling/sdk-integration.md` |

If the user wants #10 or #11 but you're building a plain **agent**, that's the signal it actually needs a **workflow** (or a workflow-agent) — revisit the piece type.

---

## After the checklist

1. For each **yes**, open the named deep-dive in the **stackbone** skill and write the code there (tools' `execute()` / workflow steps).
2. Add any declarations the table calls out: `schema.ts`, `config.schema.ts`, `agent.yaml` lines (`rag.embeddingModel`, required connectors).
3. Install only the peers you used — `eve` for an agent, `workflow` for `requestApproval`, `@ai-sdk/openai-compatible` for the model binding.
4. Boot `stackbone dev` and exercise every wired surface before calling the piece done (see the **stackbone-cli** skill).
