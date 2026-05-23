---
name: stackbone
description: >-
  Use this skill when writing the inside of a Stackbone agent template with @stackbone/sdk:
  database CRUD with Drizzle over Neon Postgres, file uploads to Cloudflare R2 via client.storage,
  LLM chat / embeddings through OpenRouter via client.ai, RAG ingest + hybrid search via client.rag,
  cross-container push jobs and schedules via client.queues, human-in-the-loop pauses via client.approval,
  reading injected secrets / config / connections via client.secrets / client.config / client.connections,
  emitting events to the org event bus via client.events, or declaring the agent's manifest in agent.yaml.
  Trigger on requests like: build an agent, add a tool to the agent, store user data, upload an image,
  call an LLM, ingest docs for RAG, pause until a human approves, send an SMS through a queued job,
  add a new entry to agent.yaml, or implement defineAgent({ invoke }) in src/index.ts.
  For CLI tasks (init, dev, publish, db migrate), use the stackbone-cli skill instead.
  For triage of errors and stuck runs, use the stackbone-debug skill.
license: MIT
metadata:
  author: stackbone
  version: '0.1.0'
  organization: Stackbone
  date: May 2026
---

# Stackbone SDK skill

This skill covers writing code **inside** a Stackbone agent template — what the creator builds with `@stackbone/sdk` and declares in `agent.yaml`. For everything around the agent (scaffolding, dev emulator, publishing, db migrations) use the **stackbone-cli** skill.

## The mental model in three sentences

- A creator writes a single file, `src/index.ts`, exporting `defineAgent({ invoke })`. Stackbone's platform-managed wrapper (`stackbone serve`) mounts it against the canonical contract — `POST /invoke`, `GET /health`, `GET /schema` — so the creator never writes HTTP code, never picks a port, never adds a Dockerfile.
- The agent runs as a Docker container on Fly Machines, one container **per agent instance** (an `agent`, not an `agent_template`). Each instance has its own dedicated Neon Postgres, its own R2 bucket, its own QStash signing keys, and its own OpenRouter sub-key — all injected as env vars at boot.
- All persistence is Postgres. Relational data, vectors (`pgvector`), full-text (`tsvector`), KV cache (table with `expires_at`) and durable queues (table `_queue_jobs` consumed with `FOR UPDATE SKIP LOCKED`) live in the same Neon. There is no Redis, no separate vector DB, no separate KV store inside the agent.

## Quick setup

### 1. Scaffold (CLI)

Use the **stackbone-cli** skill — `stackbone init --starter <slug> <new-name>` copies the chosen starter, find-and-replaces the package name, and leaves a runnable Node project. The skill auto-installs into `.claude/skills/` of the new project.

### 2. Install the SDK

```sh
pnpm install @stackbone/sdk
```

The version range is pinned by the starter; bump only when the SDK release notes say so.

### 3. Create the client

```ts
// src/index.ts
import { createClient, defineAgent } from '@stackbone/sdk';

const client = createClient(); // no args — reads injected env vars

export default defineAgent({
  async invoke(input, ctx) {
    const { data, error } = await client.database.from('users').select().where(/* ... */).all();

    if (error) return ctx.fail(error.code, error.message);
    return ctx.ok({ users: data });
  },
});
```

> Do not import `@stackbone/sdk` in HTTP middleware, do not call `createClient()` per request, do not pass connection strings explicitly. The platform injects everything; the client is a singleton.

## Injected environment variables (read by the SDK)

| Category             | Variables                                                               | Notes                                                                      |
| -------------------- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Identity             | `AGENT_ID`, `AGENT_TEMPLATE_ID`, `ORGANIZATION_ID`, `AGENT_CONFIG`      | Stable for the life of the instance                                        |
| Persistence          | `STACKBONE_POSTGRES_URL`                                                | Neon with `pgvector` + `tsvector` + cache table + `_queue_jobs` table      |
| Storage              | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `S3_ENDPOINT`             | R2 in prod, MinIO in `stackbone dev`                                       |
| LLM                  | `OPENROUTER_API_KEY`, `OPENROUTER_BASE_URL`                             | Stackbone sub-key with monthly limit. Passthrough cost, no markup.         |
| RAG parser (opt-in)  | `LLAMA_PARSE_API_KEY`                                                   | Only injected if `rag.parser: llamaparse` in `agent.yaml`                  |
| Cross-container push | `QSTASH_TOKEN`, `QSTASH_CURRENT_SIGNING_KEY`, `QSTASH_NEXT_SIGNING_KEY` | Publisher + 2 keys for HMAC rotation                                       |
| Observability        | `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_RESOURCE_ATTRIBUTES`               | Auto-instrumented (`pg`, `aws-sdk`, `undici`)                              |
| Control plane        | `PLATFORM_API_URL`, `PLATFORM_API_KEY`                                  | Used by the SDK's facades (approval, secrets, connections, config, events) |

**Never hardcode any of these. Never ask the user for them. Never set them in `agent.yaml`.** They are platform-managed.

## Module reference

All client methods return `{ data, error }`. Examples must show both branches.

| Module                 | What it wraps                                                                                      | When to use it                                                                                                                                                                               |
| ---------------------- | -------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `client.database`      | Drizzle ORM + `postgres-js` over the agent's Neon                                                  | Any structured data the agent owns: users, jobs, messages, embeddings (with `pgvector`), full-text search (with `tsvector`). See [database/sdk-integration.md](database/sdk-integration.md). |
| `client.storage`       | `@aws-sdk/client-s3` + `_storage_objects` metadata table                                           | Files: uploads, downloads, signed URLs, listing. R2 in prod, MinIO in dev.                                                                                                                   |
| `client.ai`            | `openai` SDK with OpenRouter base URL                                                              | LLM completions, chat, embeddings, vision. 300+ models. **Stream by default for long completions.**                                                                                          |
| `client.rag`           | LlamaParse (opt-in) + chunker + embeddings via `client.ai` + `pgvector` + `tsvector` hybrid search | Document ingest + retrieval. Always pair with `agent.yaml`'s `rag.parser` declaration.                                                                                                       |
| `client.queues`        | `@upstash/qstash` publisher + Hono receiver with HMAC                                              | Async work outside the request lifecycle, scheduled jobs, cross-container fan-out.                                                                                                           |
| `client.observability` | `@opentelemetry/sdk-node` + auto-instrumentations                                                  | Already auto-wired. Use `client.observability.span(name, fn)` to add custom spans.                                                                                                           |
| `client.approval`      | Stackbone HITL queue + workflow resume                                                             | Pause a run until a human decides (approve / reject / edit payload). The pause is **durable** — the agent process can die and come back.                                                     |
| `client.secrets`       | Encrypted secret vault per agent instance                                                          | Read user-managed secrets (e.g. third-party API keys the org member added in Studio). System secrets (DATABASE*URL, AWS*\*) are env vars, not `client.secrets` reads.                        |
| `client.config`        | Dynamic `AGENT_CONFIG` jsonb the org member edits in Studio                                        | Read-only at runtime. Use for feature flags, per-install tuning, branding.                                                                                                                   |
| `client.connections`   | OAuth connections the org has authorized (Notion, GDrive, Slack, ...)                              | Get fresh OAuth tokens, listed in `agent.yaml`'s `connections:`.                                                                                                                             |
| `client.events`        | Org event bus                                                                                      | Emit a typed event from this agent that other agents in the same org can subscribe to.                                                                                                       |

## agent.yaml — the manifest

The recipe published as `agent_template`. See [agent-yaml.md](agent-yaml.md) for the full reference. Minimal example:

```yaml
apiVersion: stackbone.ai/v1
name: support-triage
runtime:
  engine: node
  entry: src/index.ts
```

A capability-rich example with RAG, HITL, queues, connections and a one-time fee:

```yaml
apiVersion: stackbone.ai/v1
name: contract-reviewer
description: Reviews contracts for risky clauses and routes them to a human approver.
runtime:
  engine: node
  entry: src/index.ts
pricing:
  model: one_time_fee
  amount_eur: 49
capabilities:
  - rag
  - hitl
  - queues
rag:
  parser: llamaparse
connections:
  - provider: gdrive
    scopes: [drive.readonly]
events:
  subscribes: [contract.uploaded]
  emits: [contract.approved, contract.rejected]
```

> **Do not declare LLM keys, database URLs, queue tokens or storage credentials.** Stackbone injects them. The manifest is for **intent** (what capabilities the agent uses) and **identity** (name, pricing, description).

## The `defineAgent` contract

```ts
export default defineAgent({
  // Required: the only entrypoint a request can hit.
  async invoke(input, ctx) {
    // input: parsed and validated against the schema you export below
    // ctx.run.id, ctx.run.trigger ('manual' | 'webhook' | 'cron' | 'event' | 'embed')
    // ctx.organizationId, ctx.agentId
    // ctx.ok(data), ctx.fail(code, message)
  },

  // Optional: the JSON schema that gates GET /schema and validates `input`.
  // Required for chat-capable agents (must accept { session_id, message }).
  schema: {
    /* JSON Schema */
  },

  // Optional: subscribe to events the org emits.
  events: {
    'contract.uploaded': async (event, ctx) => {
      /* ... */
    },
  },

  // Optional: scheduled callbacks (also expressible via QStash).
  schedules: [
    {
      cron: '0 9 * * *',
      handler: async (ctx) => {
        /* ... */
      },
    },
  ],
});
```

`GET /health` and `GET /schema` are mounted by the wrapper — the creator never writes them. `POST /invoke` is the only request path. If the agent needs to expose more operations, **collapse them into `invoke` with a discriminated `action` field on `input`** (see the multi-action starter pattern). Do not add HTTP routes — they will not be reachable from the platform.

## Patterns that matter

### Always destructure `{ data, error }`

```ts
const { data, error } = await client.database.from('users').select().all();
if (error) return ctx.fail('db_read_failed', error.message);
return ctx.ok({ users: data });
```

Skipping the `error` branch trains the agent to swallow failures. The error envelope is structured (`{ code, message, details, retryable }`) — use `code` to branch on known cases.

### Inserts take an array

```ts
await client.database.from('users').insert([{ email, name }]); // ✅
await client.database.from('users').insert({ email, name }); // ❌ rejected
```

### Storage objects need both `url` and `key`

```ts
const { data, error } = await client.storage
  .from('avatars')
  .upload(file, { contentType: 'image/png' });

// Save both: `url` for serving, `key` for delete/move/list
await client.database.from('users').update({ avatar_url: data.url, avatar_key: data.key }).where(...);
```

### Stream long LLM completions

```ts
const stream = await client.ai.chat.completions.create({
  model: 'anthropic/claude-sonnet-4.5',
  messages: [...],
  stream: true,
});

for await (const chunk of stream) {
  yield chunk.choices[0]?.delta?.content;
}
```

Non-streamed completions over ~30 s often hit the platform's invocation timeout. Stream by default for chat.

### HITL is durable, not blocking

```ts
const { data: approval, error } = await client.approval.requestAndWait({
  title: 'Approve contract clause edit',
  payload: { contractId, clauseDiff },
  approverRole: 'approver',
});
```

The agent **process can die** between `request` and `wait` — the platform persists the approval and resumes the run when the human decides. Do not hold an in-memory promise; the SDK handles continuation.

### QStash for any work > 30 s

```ts
await client.queues.publish('process-contract', { contractId });

// Receiver in the same agent — Stackbone routes the callback with signed HMAC
export const receivers = {
  'process-contract': async ({ contractId }, ctx) => {
    /* ... */
  },
};
```

Use `client.queues` for fan-out, retries with exponential backoff, dead-letter inspection. The platform survives scale-to-zero — Neon hibernates, QStash wakes the container with the signed callback.

### RAG: ingest async, retrieve sync

```ts
// Ingest is queued automatically when you call .upload(); never block invoke on it
await client.rag.from('contracts').upload(file);

// Retrieval is fast (Postgres hybrid: pgvector + tsvector)
const { data, error } = await client.rag.from('contracts').retrieve(query, { limit: 5 });
```

### `client.events` is for cross-agent fan-out within an org

```ts
await client.events.emit({ type: 'contract.approved', payload: { contractId } });
```

Other agents in the same organization that **subscribed** to `contract.approved` in their own `agent.yaml` get invoked. **Don't use it as a generic webhook** — out-of-org delivery is not in scope.

## Tier and quota awareness

Each organization has a tier (`free` / `starter` / `pro` / `team`) with a credit bundle that covers license + infra + LLM tokens. When credits run out the agent **pauses** until the next period or until upgrade. The SDK does not need to check tier explicitly — failed calls return `error.code === 'tier_quota_exceeded'` with a `nextActions` block telling the org member to upgrade. Surface that error verbatim, do not retry.

## Branch the backend for risky changes

If a code change depends on a schema migration, a new RLS policy, an event subscription change in `agent.yaml`, or anything else that could leave a deployed agent broken — work on a `stackbone dev` session first (your local installation, isolated DB / R2 / QStash) instead of testing against the cloud `agent`. See the **stackbone-cli** skill (`dev` reference) for the loop.

## SDK quick reference

| Module                 | Methods (most used)                                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `client.database`      | `.from(table).select() / .insert([rows]) / .update({...}).where(...) / .delete().where(...) / .rpc(name, args)`          |
| `client.storage`       | `.from(bucket).upload(file, opts) / .download(key) / .signedUrl(key, expiresIn) / .remove(keys[]) / .list(prefix)`       |
| `client.ai`            | `.chat.completions.create({model, messages, stream?}) / .embeddings.create({input, model}) / .images.generate({prompt})` |
| `client.rag`           | `.from(collection).upload(file) / .retrieve(query, {limit, filter}) / .delete(docId)`                                    |
| `client.queues`        | `.publish(topic, payload, {delay?, schedule?}) / .schedule(name, cron, handler)`                                         |
| `client.approval`      | `.requestAndWait({title, payload, approverRole}) / .request({...}) / .resume(approvalId, decision)`                      |
| `client.secrets`       | `.get(key)` (system secrets are env vars, not here)                                                                      |
| `client.config`        | `.get<T>(path?)` (read-only AGENT_CONFIG jsonb)                                                                          |
| `client.connections`   | `.from(provider).getToken() / .from(provider).client()` (provider-typed client, refreshed token baked in)                |
| `client.events`        | `.emit({type, payload}) / .on(type, handler)`                                                                            |
| `client.observability` | `.span(name, fn) / .log(level, msg, attrs)`                                                                              |

## Important notes

- **No HTTP framework choice**. The wrapper is Hono + Node 24 LTS. You can opt in to Bun via `runtime.engine: bun` in `agent.yaml`; the SDK and contract are runtime-agnostic.
- **No Dockerfile** is needed. The platform buildpack builds the container from `package.json` + `agent.yaml` + `src/`. If you really need one (custom system deps), declare `runtime.dockerfile: ./Dockerfile` and own the contract yourself.
- **Embed-capable agents** must declare it (`capabilities: [chat-embed]`) and accept `{ session_id, message }` in `invoke`. Embed traffic counts toward Trust Layer metrics; playground traffic doesn't.
- **Trigger types** on `ctx.run.trigger`: `'manual' | 'webhook' | 'cron' | 'event' | 'embed'`. Branch on this if behavior differs (e.g. don't run expensive RAG ingest on `embed`).
- **No `console.log` in hot paths** — use `client.observability.log()`; it flows through OTel with the right attributes (`organization_id`, `agent_id`, `invocation_id`) and is queryable from Studio.
- **The agent owns its data**. Stackbone never reads or writes the agent's Neon directly. If the platform needs something (e.g. for billing), it's tracked out-of-band against partner APIs, not by querying the agent's DB.
