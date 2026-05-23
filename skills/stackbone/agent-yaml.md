# `agent.yaml` reference

The manifest of an agent template — the file that gets frozen into the `agent_template_version` row at `stackbone publish` time. Lives at the project root, next to `package.json`.

## Minimal example

```yaml
apiVersion: stackbone.ai/v1
name: support-triage
runtime:
  engine: node
  entry: src/index.ts
```

That's enough to publish a template. Everything else is opt-in.

## Capability-rich example

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
  - database
  - storage
  - ai
  - rag
  - hitl
  - queues
  - connections
  - events

rag:
  parser: llamaparse

connections:
  - provider: gdrive
    scopes: [drive.readonly]
  - provider: notion
    scopes: [read]

events:
  subscribes:
    - contract.uploaded
  emits:
    - contract.approved
    - contract.rejected

screenshots:
  - assets/dashboard.png
  - assets/approval-flow.png

category: legal
tags: [contracts, compliance, hitl]
```

## Field reference

### `apiVersion` (required)

Always `stackbone.ai/v1`. Locks the manifest to this skill's schema; future major versions get explicit migration paths.

### `name` (required)

The template slug. Must match `[a-z0-9-]+` and be unique within the org. Lowercase, hyphens-only. Cannot be changed after first publish (it's the slug a creator deeplinks).

### `description` (optional, recommended)

One-paragraph human description. Shown on the marketplace listing. ≤ 280 characters renders better in catalog cards.

### `runtime` (required)

```yaml
runtime:
  engine: node # node (default) | bun
  entry: src/index.ts # path to defineAgent({ invoke }) export
  dockerfile: ./Dockerfile # optional escape hatch — see below
```

- **`engine: node`** uses Node 24 LTS via the platform buildpack. Hot reload in `stackbone dev` and standard wrapper mounting.
- **`engine: bun`** opts into Bun 1.x — same wrapper, faster cold start in some workloads. Worth trying only after Node works.
- **`dockerfile:`** — escape hatch. You own the container, the contract (`/invoke`, `/health`, `/schema`) and the wrapper mounting. **Lose hot reload, lose auto-injected env vars verification, lose buildpack CVE scanning consistency.** Only use for system-deps the buildpack doesn't cover (custom OCR binaries, native extensions).

### `pricing` (optional, defaults to free)

```yaml
pricing:
  model: one_time_fee
  amount_eur: 49
```

MVP supports only `one_time_fee` and free (omit the block). Subscription and pay-per-invocation arrive in V1+ — those rows will be rejected at publish time today.

### `capabilities` (recommended)

Declares which SDK modules the agent uses. Maps 1:1 to `client.*` modules. The platform uses this for:

- Injecting only the env vars the agent needs (don't inject `LLAMA_PARSE_API_KEY` if the agent doesn't list `rag`).
- Computing tier compatibility (e.g. `connections` is a tier-gated capability).
- Showing the right buttons in Studio (no HITL inbox link if `hitl` isn't declared).

Available capabilities: `database`, `storage`, `ai`, `rag`, `hitl`, `queues`, `connections`, `events`, `chat-embed`, `secrets`, `config`.

> If the code uses a capability not declared here, the SDK returns `error.code = 'capability_not_granted'` at runtime. This is deliberate — it prevents accidentally bloating an install's permission scope.

### `rag` (required when `capabilities` includes `rag`)

```yaml
rag:
  parser: llamaparse # llamaparse | none
```

- **`llamaparse`** opts into the LlamaParse-managed parser for PDFs/Office/images with OCR. Injects `LLAMA_PARSE_API_KEY`, counts against the org's parse quota.
- **`none`** uses the chunker only — the agent provides text. No LlamaParse cost.

### `connections` (required when `capabilities` includes `connections`)

```yaml
connections:
  - provider: gdrive
    scopes: [drive.readonly]
  - provider: notion
    scopes: [read]
```

Each entry is an OAuth provider the agent will call via `client.connections.from(<provider>).client()`. The org member must authorize each one in Studio's Connections tab **before** installing the agent (the install flow prompts for missing authorizations).

Supported providers: `gdrive`, `notion`, `slack`, `gmail`, `github`, `linear`, `hubspot`, `salesforce`. Each provider's available scopes are listed in `stackbone docs connections`.

### `events` (required when `capabilities` includes `events`)

```yaml
events:
  subscribes:
    - contract.uploaded
  emits:
    - contract.approved
    - contract.rejected
```

- **`subscribes`** — event types this agent listens to. The org's event bus fan-outs to every agent that subscribes. The agent receives them in `defineAgent({ events: { 'contract.uploaded': handler } })`.
- **`emits`** — event types this agent produces. The platform uses this for the visual fan-out graph in Studio (which agents talk to which).

Event types are arbitrary strings; convention is `<noun>.<past-tense-verb>` (e.g. `invoice.created`, `lead.qualified`).

### `chat-embed` capability

Set this if the agent will be embedded as a chat widget on a third-party site via `agent_embed`. Requires:

- `defineAgent({ invoke })` to accept `{ session_id: string, message: string }`.
- The agent to own its conversation state (Neon table).

The marketplace lists the agent as chat-embeddable; the org member can create one `agent_embed` per install and embed it on their domain via `<script>` tag.

### `screenshots` (optional)

```yaml
screenshots:
  - assets/dashboard.png
  - assets/approval-flow.png
```

Paths relative to the project root. Bundled into the template artifact, served by the marketplace's CDN. Recommended: 1280×720 PNG/JPG, ≤ 500 KB each.

### `category` / `tags` (optional)

Used for marketplace filtering. Free-form; the platform reconciles them against the canonical taxonomy.

## What you can't put here

- **LLM API keys** (`OPENROUTER_API_KEY`, anything else). Stackbone provisions per-install sub-keys; the creator does not pick the model gateway.
- **Database URLs**, **S3 credentials**, **QStash tokens**. Same reason — platform-injected.
- **Custom `port:`**. The wrapper binds to whatever Fly assigns and proxies; the agent code never sees the port.
- **`replicas:`**, **`memory:`**, **`cpu:`**. Resource sizing is tier-driven at install time, not template-declared.
- **`min-sdk-version`** (yet). Today the buildpack pins the SDK by the project's `package.json` dependency. A `requires.sdk` field is on the roadmap.

## Validation

`stackbone publish` validates `agent.yaml` against the schema before any network I/O. To validate without publishing:

```sh
stackbone docs agent-yaml --validate
```

(Reads the local `agent.yaml`, returns `{ valid: true }` or the list of errors with field paths.)
