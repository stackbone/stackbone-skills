# Workspace + agent manifests

A Stackbone project is a **workspace** with two manifest layers:

- **Convention scan (primary).** The workspace registry is **derived from the files on disk** — there is no hand-maintained list to keep in sync. `stackbone dev` and `stackbone publish` both read this same convention. Agents are every `deep-agents/<name>/` folder with an `index.ts` (default-exporting `defineDeepAgent(...)`); workflows are every `workflows/<name>.workflow.ts`. Most projects need no config file at all.
- **`stackbone.config.ts` (optional override, workflows only).** A file that default-exports `defineWorkspace({ agents: [], workflows })`. If present, its **workflow** list wins over the workflow scan; deep agents are **always** discovered from their folders and cannot be declared here. Reach for it only when you need to override the workflow scan; see the **stackbone** skill for the full shape.
- **`agent.yaml`** — an **OPTIONAL workspace manifest.** It is **not** the discovery marker and is **not** scaffolded by default. When present at the workspace root, `stackbone dev` reads only `database.schema` / `database.migrations` and `dev.autoMigrate` from it; the richer blocks below (runtime, rag, connections, automations, protocol) apply to the classic single-agent manifest and the publish-time build. The schema is `.strict()`.

This page is the field reference for `agent.yaml` when you do write one.

## Discovery by convention

The workspace shape comes straight from the directory layout:

- **Agents** — every `deep-agents/<name>/` folder containing an `index.ts`. The folder basename is the agent's name; there is no per-agent `package.json` and no name field.
- **Workflows** — every `workflows/<name>.workflow.ts`. The workflow name is the file basename without the `.workflow.ts` suffix; the exported function is `<camelCase(name)>Workflow` (e.g. `qualify-lead.workflow.ts` exports `qualifyLeadWorkflow`).

You scaffold these pieces with `stackbone init` (workspace shell + an optional first piece) and `stackbone add` (each new deep-agent / workflow / workflow-agent) — see the **stackbone-cli** skill. `add` only ever writes new files; it never edits your existing TypeScript and never edits `stackbone.config.ts`.

## `stackbone.config.ts` — the optional workflow override

```ts
import { defineWorkspace } from '@stackbone/sdk';

export default defineWorkspace({
  agents: [],
  workflows: [
    {
      name: 'onboarding',
      module: 'workflows/onboarding.workflow.ts',
      export: 'onboardingWorkflow',
    },
  ],
});
```

- **`agents[]`** — legacy, stays empty: deep agents are always discovered from their `deep-agents/<name>/` folders.
- **`workflows[]`** — `{ name, module, export }`. `name` is how you address/trigger the workflow; `module` is the `*.workflow.ts` path; `export` is the exported function name inside it.

When this file is absent — the common case — the workflow registry is inferred from the convention scan above, so you only author `stackbone.config.ts` to override that inference.

A workflow declares its input/output **by convention** — sibling `inputSchema` / `outputSchema` exports next to the `'use workflow'` function (see the **stackbone** skill), not in this config. The build harvests them.

---

## `agent.yaml` — the per-agent manifest

The manifest of one agent. In a classic single-agent project it lives at the project root; in a workspace, an optional root-level `agent.yaml` only feeds the database/dev keys (see above).

> **The schema is `.strict()`.** Any key not listed below makes `stackbone dev` / `stackbone publish` **fail parse** with `Unrecognized key(s)`. Marketplace metadata (description, pricing, category, screenshots) is set in Studio / the create-template flow, **not** here — putting those keys in `agent.yaml` is a parse error, not a no-op.

### Minimal example

```yaml
apiVersion: stackbone.ai/v1
name: support-triage
runtime:
  engine: node
  entry: src/index.ts
```

That's enough to publish — every other field has a default. With zero fields beyond `apiVersion` + `name`, the runtime is `node` / `src/index.ts`, the schema is `./src/schema.ts`, migrations live under `./.stackbone/migrations`, `dev.autoMigrate` is on, and `rag.embeddingModel` defaults to `openai/text-embedding-3-small`. The `connections` and `automations` blocks default to empty.

### Full example

```yaml
apiVersion: stackbone.ai/v1
name: contract-reviewer
version: v0.2.0 # optional display label, copied verbatim into the build row

runtime:
  engine: node # node (default) | bun
  entry: src/index.ts # path to the agent entry

database:
  schema: ./src/schema.ts # your Drizzle pgTable definitions
  migrations: ./.stackbone/migrations # where db migrate create writes

dev:
  autoMigrate: true # stackbone dev applies + auto-generates migrations on boot

rag:
  embeddingModel: openai/text-embedding-3-small # only knob rag has now

connections:
  required: [telegram] # connector accounts the install must connect before use

automations:
  recipes:
    - key: reply-on-message # per-install idempotency slug
      name: Reply to inbound Telegram messages # human label
      trigger:
        connector: telegram
        trigger: message-received
      handler: onMessage # the named entry the event routes to
      inputMapping: '{ "chatId": message.chat.id, "text": message.text }' # optional JSONata
      action: # optional — omit for a trigger-only recipe
        connector: telegram
        action: send-message

protocol:
  required: 1 # minimum Stackbone Agent Protocol contract version (optional floor)
```

## Field reference

### `apiVersion` (required)

Always the literal `stackbone.ai/v1`. Locks the manifest to this schema; future major versions get explicit migration paths.

### `name` (required)

The agent name. Lowercase, hyphens-only convention. Inside a workspace the discovery scan keys agents on their `deep-agents/<name>/` folder basename, **not** on this file — `agent.yaml` never names a workspace agent.

### `version` (optional)

A free-form display label (e.g. `v0.2.0`), max 64 chars, copied verbatim into the matching build row and shown on the install detail page. **Never validated as semver, never unique-constrained** — pure display. The actual version bump at publish time is chosen by `stackbone publish` (see the **stackbone-cli** skill).

### `runtime` (optional, defaults to `{ engine: node, entry: src/index.ts }`)

```yaml
runtime:
  engine: node # node (default) | bun
  entry: src/index.ts
```

- **`engine: node`** uses Node 24 LTS via the platform buildpack. Hot reload in `stackbone dev`.
- **`engine: bun`** is in the enum but **deferred** — the CLI rejects it today with a friendly "set `runtime.engine` to `node`" message. Use `node`.
- **`customDockerfile`, `systemDeps`, `buildSecrets`, `packageManager`** are declared in the schema only to give a precise rejection message — they are **not supported** today and each errors with its own hint. The buildpack detects the package manager from the lockfile; there is no custom Dockerfile escape hatch yet.

### `database` (optional, has defaults)

```yaml
database:
  schema: ./src/schema.ts # path to your Drizzle pgTable definitions
  migrations: ./.stackbone/migrations # path the CLI reads/writes migrations
```

Path strings are resolved by the CLI at command time relative to the agent root — the manifest stays declarative. See the **stackbone-cli** skill (`db` reference) for the migration flow.

### `dev` (optional, defaults to `{ autoMigrate: true }`)

```yaml
dev:
  autoMigrate: true
```

When `true`, `stackbone dev` applies pending migrations on boot **and** auto-generates + applies a new one whenever `schema.ts` changes. Set `false` to drive migrations yourself.

### `rag` (optional, defaults to `{ embeddingModel: openai/text-embedding-3-small }`)

```yaml
rag:
  embeddingModel: openai/text-embedding-3-small
```

`rag` configures **only** the embedding model. The RAG tables (`rag_chunks`, the `pgvector` HNSW index) are **provisioned by the platform on every install** — there is no per-agent setup step and no `parser:` key. The block is `.strict()`, so a stale `parser:` fails parse. Agents that never touch `stackbone.rag` can omit the block entirely.

### `connections` (optional, defaults to `{ required: [] }`)

```yaml
connections:
  required: [telegram, gmail] # connector ids the agent needs
```

`required` lists the **connector accounts** the agent depends on. At install time the flow prompts the operator to connect any of these the workspace hasn't connected yet, so the agent's `stackbone.connection(...)` calls have a connection to resolve at runtime. The ids are catalog connector ids — the same ones `stackbone connectors` lists; the format is validated here, while whether an id actually exists is checked by the control plane at publish/install time. Agents that call no connector omit the block. See the **stackbone** skill's connections doc and the **stackbone-cli** skill's `connectors` reference.

### `automations` (optional, defaults to `{ recipes: [] }`)

```yaml
automations:
  recipes:
    - key: reply-on-message # per-install idempotency slug
      name: Reply to inbound messages # human label, 1–200 chars
      trigger:
        connector: telegram
        trigger: message-received # an event id on that connector
      handler: onMessage # the named entry the event routes to
      inputMapping: '{ "chatId": message.chat.id, "text": message.text }' # optional JSONata
      outputMapping: '{ "ok": reply.delivered }' # optional JSONata
      action: # optional — omit for a trigger-only recipe
        connector: telegram
        action: send-message
```

Each recipe is a ready-made automation the **install flow seeds** as an ordinary, editable automation (the operator can change it afterwards). It mirrors the pipeline `trigger → inputMapping → handler → outputMapping → action`:

- **`key`** — a slug; the per-install idempotency key the seeder upserts on, so re-publishing updates the same automation instead of duplicating it.
- **`name`** — human label (1–200 chars) shown in the install's automations list.
- **`trigger`** — `{ connector, trigger }`: the catalog event that fires the recipe.
- **`handler`** — the named entry the event routes to (for an agent, the routing target is the agent itself).
- **`inputMapping` / `outputMapping`** — optional JSONata expressions applied before and after the handler (the same engine the runtime applies in production).
- **`action`** — optional `{ connector, action }`: a connector action run with the handler's (mapped) output. Omit it for a trigger-only recipe.

Every block is `.strict()` — a mistyped key is a loud parse error.

### `protocol` (optional, no default)

```yaml
protocol:
  required: 1
```

An optional floor on the negotiated Stackbone Agent Protocol contract version. When the negotiated contract version is below `protocol.required`, every gated surface call returns `contract_version_unsupported`. Omit it to fall back to the SDK's implicit minimum. **This is the only capability/protocol knob in the manifest** — capability gating is negotiated through the contract handshake, not declared as a field.

## What you can't put here

These keys are **not** part of the on-disk manifest. Some are marketplace metadata set elsewhere, some are deferred — either way, the `.strict()` schema rejects them at parse time:

- **`description`, `pricing`, `category`, `tags`, `screenshots`** — marketplace listing metadata. Set in Studio, not `agent.yaml`.
- **`capabilities:`** — there is no capability list in the manifest. Surface availability is gated by the negotiated contract (see `protocol` above), not declared here.
- **`events:`** — no event-bus surface ships, so there is no manifest field for it. (`connections:` and `automations:` **are** valid blocks now — see above.)
- **`rag.parser`** — removed; `rag` only has `embeddingModel`.
- **`runtime.dockerfile` / `runtime.systemDeps` / `runtime.buildSecrets` / `runtime.packageManager`** — deferred; each errors with its own message.
- **LLM keys, database URLs, S3 credentials** — platform-injected, never declared. (See the injected-env table in the **stackbone** skill.)
- **`port:` / `replicas:` / `memory:` / `cpu:`** — the runtime binds the port; resource sizing is tier-driven at install time.

## Validation

`stackbone publish` validates the manifest against the schema before any network I/O (there is **no** `--dry-run` flag — `publish` already writes only a local, digest-verifiable bundle and pushes nothing). `stackbone dev` also parses the manifest on boot, so a schema error surfaces the moment you start the emulator. See the **stackbone-cli** skill for both.
