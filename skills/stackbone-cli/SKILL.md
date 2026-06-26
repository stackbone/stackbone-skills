---
name: stackbone-cli
description: >-
  Use this skill when driving the `stackbone` CLI to scaffold, develop, publish or operate an agent on Stackbone:
  authenticating with device flow (login / whoami), scaffolding a workspace shell offline (init, optionally with a
  first agent / workflow / workflow-agent piece), adding more pieces into an existing workspace
  (add agent / add workflow / add workflow-agent), linking an existing directory to an agent (link),
  running the local dev stack with Studio (dev),
  publishing the workspace (publish), switching the active organization the CLI acts as
  (organization use), managing database migrations with Drizzle
  (db migrate up / create / status), reading workspace metadata in JSON (metadata),
  listing the available connectors and the account's connections with ids and names (connectors),
  printing inline docs (docs), and operating a running agent installation from the shell —
  discovering install targets (agents), inspecting and triggering the durable workflows it exposes (workflows list / schema / start),
  inspecting and controlling runs (runs), tailing logs (logs),
  querying/browsing the agent DB (db query / schemas / table),
  moving objects in the store (storage), operating the managed RAG index (rag), setting secrets (secrets),
  versioning the config document and regenerating its types (config), inspecting the protocol contract (contract),
  deciding human-in-the-loop approvals (hitl), managing the prompt catalog (prompts),
  and reading the OpenRouter wiring (openrouter).
  Trigger on requests like: scaffold a workspace, init an empty project, add an agent to my workspace,
  add a workflow, add a workflow that calls an agent (workflow-agent), start the dev stack,
  publish my workspace, generate a migration, list my orgs, switch organization, list connectors or connections,
  see what's linked, log in to Stackbone, list my agents,
  list the workflows an install exposes, start a workflow by name, retry a failed run, tail the logs, run a SELECT against the agent,
  set a secret, roll the config back, approve a HITL request.
  For writing code inside the agent with @stackbone/sdk, use the stackbone skill instead.
  For triage of errors and stuck runs, use the stackbone-debug skill.
license: MIT
metadata:
  author: stackbone
  version: '0.4.1'
  organization: Stackbone
  date: June 2026
---

# Stackbone CLI

Command-line tool for scaffolding, developing, publishing and operating Stackbone agents.

A Stackbone project is a **workspace**: one or more durable [eve](https://eve.dev/docs/introduction) agents (each `agents/<name>/agent.yaml`) plus durable [Workflow SDK](https://workflow-sdk.dev/docs) workflows (each `workflows/<name>.workflow.ts`). The workspace is **derived by convention** from those files on disk — any `agents/<name>/agent.yaml` or `workflows/<name>.workflow.ts` makes the directory a workspace. A `stackbone.config.ts` (default-exporting `defineWorkspace({ agents, workflows })`) is an **optional override**: if present it wins over the convention scan, but most projects need none. The CLI scaffolds it (`init`, `add`), runs the whole stack locally (`dev`), packages it (`publish`), and operates a running install (runs, workflows, hitl, logs, db, …).

## Critical: how to call the CLI

The binary is `stackbone` (installed into your project's `node_modules/.bin/` by `package.json` after `npm install` / `pnpm install`, and globally available if you install `@stackbone/cli` yourself). **Always invoke it explicitly**:

```sh
stackbone <command> [--json] [--yes] [--verbose]
```

You can also run it one-off without installing: `npx @stackbone/cli@alpha <command>`. If `stackbone --help` fails, you have not run `npm install` in the project or have not installed the global CLI.

**Session start** — verify auth and project:

```sh
stackbone whoami    # who am I, which org
stackbone current   # is this directory linked to an agent?
```

If not authenticated: `stackbone login` (opens browser; falls back to device-flow code if no browser).
If no project yet: `stackbone init` (emit a workspace shell, offline by default) or `stackbone link` (attach an existing directory to an agent).

## Global options

| Flag           | Description                                                                                                                                                                                                                                                                                                                                                                                                                   |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--json`       | Emit a versioned JSON envelope instead of human-formatted text. Success writes `{ schema_version, …payload }` to stdout (the payload fields are spread at the top level, not nested under `data`); an error writes `{ schema_version, error: { code, message, suggestion? }, exit_code }` to stderr. Also **skips value-collection prompts** — errors out if required params are missing. (`STACKBONE_JSON=1` is equivalent.) |
| `--yes` / `-y` | Auto-accept Y/N confirmation prompts (deletes, overwrites). Does NOT skip value prompts — combine with `--json` for fully non-interactive runs.                                                                                                                                                                                                                                                                               |
| `--verbose`    | Stream every log line instead of spinner UI. Useful for CI / agent shells where TTY redraws break parsing.                                                                                                                                                                                                                                                                                                                    |

> When an AI coding agent is the consumer, pass `--json --yes` by default. The confirm prompts protect humans typing `! stackbone …`, not autonomous agents.

## Exit codes

| Code | Meaning                                                             |
| ---- | ------------------------------------------------------------------- |
| 0    | Success                                                             |
| 1    | Generic error (network, validation, backend 4xx/5xx)                |
| 2    | Not authenticated                                                   |
| 3    | Project not linked (no `.stackbone/project.json` in this directory) |
| 4    | Resource not found                                                  |
| 5    | Permission denied (RBAC, capability, tier quota)                    |

Branching on exit codes lets the agent recover deterministically — e.g., `exit 3` → run `stackbone init` or `stackbone link` first.

## Environment variables

| Variable              | Description                                                                           |
| --------------------- | ------------------------------------------------------------------------------------- |
| `STACKBONE_API_URL`   | Override the control plane URL (dev / staging). Highest-priority source for `apiUrl`. |
| `STACKBONE_JSON`      | `1` forces JSON-envelope output, exactly like passing `--json`.                       |
| `STACKBONE_VERBOSE`   | `1` streams every log line, like `--verbose`.                                         |
| `STACKBONE_LOG_LEVEL` | Pino log level for the CLI's own diagnostics (to stderr).                             |

> The CLI does **not** read access-token / org-id / agent-template-id env overrides. Authentication comes only from `~/.stackbone/credentials.json` (minted by `stackbone login`); the active org comes from `stackbone organization use`; the linked agent from `.stackbone/project.json`.

---

## Commands

### Auth

- `stackbone login` — RFC 8628 device flow. Browser opens; copy the code if it doesn't. See [references/login.md](references/login.md)
- `stackbone logout` — clear stored credentials at `~/.stackbone/credentials.json`
- `stackbone whoami` — show the active user and organization

### Project lifecycle

- `stackbone init [dir] [--with empty|agent|workflow|workflow-agent] [--name <ws>]` — emit a **workspace shell** and, optionally, a first piece on top of it. **Workspace-first and offline by default**: with `--with empty` (the default) it writes only the shell — an `agents/` folder (one eve agent per subfolder), a `workflows/` folder, `package.json`, `tsconfig.json`, an `.npmrc` (eve needs a hoisted `node_modules`), `.gitignore`, a README and the coding-agent skills (best-effort) — and makes **no network call**. `--with` picks an optional first piece: `empty` (shell only, fully offline), `agent`, `workflow` or `workflow-agent`; with a TTY and no `--with`, it shows an interactive picker. Only the agent-creating kinds (`agent`, `workflow-agent`) touch the network — they register the agent eagerly in the control plane, so **you must be signed in** (`--with empty` and `--with workflow` are fully offline). `--name` sets the workspace name (and the default first-piece name); the `[dir]` positional sets the target subdirectory. There is **no** `--starter` / `--template` on `init` anymore — passing one prints a migration message and exits non-zero (templates are now per-piece on `add`). See [references/init.md](references/init.md)
- `stackbone add agent <name> [--template <t>]` — add one eve agent folder under `agents/<name>/` and register it eagerly in the control plane + create its own local-dev install (so **you must be signed in**). Idempotent: re-running with the same `<name>` resolves the same control-plane row.
- `stackbone add workflow <name> [--template <t>] [--calls <agent>]` — add one durable workflow file at `workflows/<name>.workflow.ts`. **Never** touches the control plane (workflows are dev-only today) — no login required. `--calls <agent>` wires a step that delegates a turn to that agent (the workflow → agent hybrid).
- `stackbone add workflow-agent <name> [--template <t>]` — the composed template: scaffold an agent **and** a workflow already wired to call it (the qualify-lead → lead-qualifier pattern). Registers the agent like `add agent`, so **you must be signed in**.
- `stackbone link` — attach the current directory to an existing agent (writes `.stackbone/project.json`)
- `stackbone current` — show the authenticated user and the linked agent
- `stackbone list` — list organizations the user owns or is a member of, with their agents
- `stackbone organization use [slug]` — choose which organization the CLI acts as (the active-org context `init`, `link`, `dev` and the agent-runtime surfaces resolve against). Pass the slug to switch directly; omit it to pick interactively from your memberships (the current one is marked). Non-interactive (`--json` or CI) **requires** the slug — discover slugs with `stackbone list`. `--json` emits `{ organization: { id, slug, name } }`. Exit `4` when the slug is not one of your memberships.
- `stackbone metadata` — machine-readable overview of the workspace (auth status, linked project, capabilities, current agent/template ids). **Run this first** with `--json` to discover state before building features.

> **`add` only writes new files.** It never edits your existing TypeScript and never edits `stackbone.config.ts`. A name collision fails with a clear error (re-run with `--force` to overwrite). `add` must run inside a workspace; outside one it exits `3` (`no_project`). The agent-creating kinds (`add agent`, `add workflow-agent`) register eagerly in the control plane, so they exit `2` (`auth`) when you are not signed in; `add workflow` is fully offline.

> **The workspace registry is derived by convention** — from the files on disk, not a hand-maintained list. Agents are every `agents/<name>/agent.yaml` (the manifest `name:` must equal the folder basename); workflows are every `workflows/<name>.workflow.ts` (the workflow name is the file basename without `.workflow.ts`, and its exported function is `<camelCase(name)>Workflow` — e.g. `qualify-lead.workflow.ts` exports `qualifyLeadWorkflow`). `stackbone dev` and `stackbone publish` both read this same convention; a `stackbone.config.ts` only overrides it when present.

### Connectors & connections — `stackbone connectors`

- `stackbone connectors` — list the available connectors (the catalog: ids + `authKind` + action/trigger ids) **and** the connections that exist for the account, nested under each connector with their **id, name and health**. This is the discovery step before you call a connection from agent or workflow code with `stackbone.connection('<id or unique name>')`: it gives you the id/name to pass. Read-only, authenticated. Listing connections needs the owner/admin `connections:manage` capability; without it the command still prints the catalog and flags `connections_unavailable`. See [references/connectors.md](references/connectors.md).

```sh
stackbone connectors          # human table: connectors + nested connections
stackbone connectors --json   # { connectors: [{ id, authKind, actions, triggers, connections: [{ id, name, healthStatus }] }] }
```

> When an account has **several** connections of the same connector, address the one you mean by its id or unique name: `stackbone.connection('<id or unique name>')`. Run `stackbone connectors --json` to get the id/name to pass. (Connector authoring + the `@stackbone/sdk/connect` surface live in the **stackbone** skill.)

### Local dev — `stackbone dev`

Boot the whole workspace locally with hot reload and Stackbone Studio. The single HTTP server (with Studio mounted) listens on **`http://127.0.0.1:4242`** by default and binds to loopback only. It brings up the local stack — Postgres, Redis and MinIO — and runs the same contract the durable runtime serves in production, so there are no surprises at publish time. See [references/dev.md](references/dev.md).

```sh
stackbone dev                 # server + Studio on 127.0.0.1:4242
stackbone dev --port 4300     # custom port
stackbone dev --listen        # bind 0.0.0.0 so it's reachable from your LAN
stackbone dev --verbose       # stream every log line instead of the spinner UI
```

The agents boot as subprocesses on free ports behind that server; you reach them through the front-door chat route (`POST /api/agents/:name/chat`) and the workflow routes (`POST /api/workflows/:name/start`), not a separate `:3000`. Saving anything under your agent/workflow source hot-reloads the affected agent.

### Workflows — `stackbone workflows`

Inspect the durable [Workflow SDK](https://workflow-sdk.dev/docs) workflows the targeted installation exposes, the input/output JSON Schema each one declares (recovered from the sibling `inputSchema` / `outputSchema` Zod exports a workflow file ships), and **start one by name**. Targets the local-dev install by default; override with `--agent <id>`. See [references/workflows.md](references/workflows.md).

```sh
stackbone workflows list --json                                       # { items: [{ name, trigger, hasSchema }] } — ◆ = declares a schema
stackbone workflows schema onboarding --json                          # { schema: { input, output } } — each half null when undeclared
stackbone workflows start onboarding --input '{"userId":"u1"}' --json # start a run → { workflowName, runId }
```

> `list` / `schema` are read-only (no `--yes`); `start` triggers a run by name — the name resolves server-side, so the workflow need not be bundled in the CLI, and input comes from `--input <json>` or `--input-file <path>`. From inside agent/workflow code the SDK equivalents are `stackbone.workflows.start` / `stackbone.workflows.startAndWait` on the ambient client (see the **stackbone** skill). Dev-first: the `stackbone dev` emulator serves these today; a cloud install's workflow routes may still 404 until the prod port lands.

### Publish — `stackbone publish`

Package the current workspace. Detected by convention — any `agents/<name>/agent.yaml` (or an explicit `stackbone.config.ts` override) — `publish` compiles every eve agent + every workflow on this host and packs them into `dist/stackbone/workspace-bundle.tar`, writing a `workspace-bundle.json` pointer beside it with the digest, sizes and contents. Native dependencies (`.node` add-ons) are rejected up-front — only pure JS runs in the runtime image. See [references/publish.md](references/publish.md).

```sh
stackbone publish            # build every agent + workflow, pack the bundle tar
stackbone publish --json     # emit a JSON envelope (digest, sizes, agents, workflows)
```

> There is **no** `--version`, `--dry-run` or `--yes` on `publish`. The bundle is written locally and verifiable by digest; the upload to object storage + the build-pointer registration are the platform's provisioning job on deploy — the CLI produces the artifact, it does not push it.

### Database — `stackbone db`

Manage Drizzle migrations **and** browse the agent's dedicated Neon read-only from the shell. See [references/db.md](references/db.md).

Migration verbs (drizzle-native; run against `STACKBONE_POSTGRES_URL`, which `stackbone dev` exports):

- `stackbone db migrate create <name>` — diff `src/schema.ts` against the journal and write a new SQL migration under `.stackbone/migrations/`
- `stackbone db migrate up [--target <tag>]` — apply every pending migration (advisory lock + journal table; safe to re-run)
- `stackbone db migrate status` — classify each migration as applied / pending / drifted

Read-only explorer verbs (HTTP, target one installation; default local-dev install, override with `--agent <id>`):

- `stackbone db query <sql>` — run an ad-hoc **single SELECT** (SQL from the positional, `--file <path>`, or stdin); the backend rejects anything that isn't a read, and rows truncate at 1000
- `stackbone db schemas` — list the schemas and tables visible to the install, with row estimates
- `stackbone db table <schema> <table>` — browse one table with cursor pagination (`--limit --cursor --order`)

> Migrations run inside a backend-managed transaction. Do not put `BEGIN`, `COMMIT` or `ROLLBACK` in your migration files; the journal lives in the agent's Neon. The RAG schema is platform-provisioned per install (no `db add-rag` — never hand-write it). Schema changes go through `db migrate create`, not `db query` (which is read-only).

### Inline docs — `stackbone docs`

- `stackbone docs` — list available topics
- `stackbone docs sdk` — full `@stackbone/sdk` reference
- `stackbone docs cli` — this command surface in long form
- `stackbone docs agent-yaml` — the `agent.yaml` manifest reference
- `stackbone docs <topic> --json` — structured output for agent parsing

> Prefer `stackbone docs` over web search when the user asks "how does the SDK do X" — the inline docs ship with the installed CLI and match the installed SDK version.

---

## Non-obvious behaviors

**Authentication is device flow, not OAuth redirect.** `stackbone login` opens the browser to a code-entry page. The CLI polls the platform until the code is approved. If the browser fails to open, the CLI prints the URL and code in plain text — copy-paste them into any browser on any machine.

**Credentials are local-only, chmod 600.** Stored at `~/.stackbone/credentials.json`. Never committed, never shipped in containers. At runtime the agent uses the env vars the runtime injects (`HMAC_SECRET`, `STACKBONE_SECRET_KEY`, `STACKBONE_INSTALLATION_ID`, `DATABASE_URL`, …), not your personal session.

**`.stackbone/project.json` is the link.** Generated by `init` or `link`, contains `{ organizationId, agentTemplateId, localDevInstallationId? }`. Add `.stackbone/` to `.gitignore` — `init` does this automatically.

**`stackbone init` is workspace-first.** It emits the workspace shell (offline by default with `--with empty`); the `[dir]` positional sets the target subdirectory and `--name` sets the workspace name (slugified). A bare `stackbone init` writes into a subdirectory derived from the workspace name so it doesn't pollute the current directory; `stackbone init my-thing` writes into `./my-thing/`. Only `--with agent` / `--with workflow-agent` touch the network (eager registration).

**`stackbone dev` boots its own local stack.** Postgres, Redis and MinIO come up automatically; the agents run as subprocesses on free ports behind the `:4242` server. Restarting your shell leaves the data services running, so the next `stackbone dev` picks up where it left off.

**`stackbone publish` rebuilds every time.** A workspace (detected by convention — any `agents/<name>/agent.yaml`, or an explicit `stackbone.config.ts`) recompiles every agent + workflow on this host and re-packs the tar; the digest only changes when the inputs do, so an unchanged workspace yields a stable `sha256:` you can compare. There is no `--cache` flag.

**`stackbone db migrate up` is idempotent.** Re-running after partial failure picks up where it left off — the journal table records the last applied version. Never edit a migration file after it has been applied; create a new one instead.

**Targeting a cloud agent vs the local-dev install.** The agent-runtime command groups (`workflows`, `runs`, `logs`, `db query`/`schemas`/`table`, `storage`, `rag`, `secrets`, `config`, `contract`, `hitl`, `prompts`, `openrouter`) default to the **local-dev installation** if one is active (`.stackbone/project.json.localDevInstallationId`); to target a cloud `agent`, pass `--agent <id>` per invocation. When neither resolves they exit `3` (`no_project`) with a suggestion to pass `--agent` or run `stackbone dev`. There is **no** `stackbone use <id>` that persists a target — that hidden state was rejected because it's the exact foot-gun the safety design avoids. To discover the install ids/slugs to pass as `--agent`, run `stackbone agents list --json` (the `agents` group is the target selector, so it takes no `--agent` of its own).

**Tier quota is enforced server-side.** If the org's credit bundle is spent, mutating commands return exit code 5 (permission). The CLI does **not** emit an `error.code` of `tier_quota_exceeded`, and there is no `nextActions` field — the JSON error envelope carries `{ code, message, suggestion? }`. (`tier_quota_exceeded` is a string on the control plane's HTTP 402 _body_, not a CLI/SDK error code — see the stackbone-debug skill.) Surface the message verbatim; do not retry.

---

## Agent-runtime surfaces

These command groups **operate a running agent installation** (cloud or the local-dev install). Every verb accepts `--json`, targets the active install or a `--agent <id>` (except `agents`, which IS the target selector and takes no `--agent`), and any **destructive** verb requires `--yes` (without it the verb exits `5`/permission before any network call). Pagination is uniform: paginated lists take `--limit` + `--cursor` and emit `{ items, nextCursor, prevCursor }` plus any domain extras. Per-command flags, outputs and exit codes live in the `references/` files linked in each row.

| Group                  | Verbs (positional args; ✱ = destructive, needs `--yes`)                                                                                                                                                                                           | Reference                                            |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `stackbone agents`     | `list`, `get <agentSlug>` — discover install targets; no `--agent` flag                                                                                                                                                                           | [references/agents.md](references/agents.md)         |
| `stackbone workflows`  | `list`, `schema <name>`, `start <name> [--input/--input-file]` — inspect the durable workflows the install exposes (+ each one's input/output schema) and start one by name                                                                       | [references/workflows.md](references/workflows.md)   |
| `stackbone runs`       | `list [--status --limit --cursor]`, `get <runId>`, `retry <runId>`✱, `cancel <runId>`✱                                                                                                                                                            | [references/runs.md](references/runs.md)             |
| `stackbone logs`       | `tail [--run --level --q --trace-id --since --until --follow --limit]` — SSE stream                                                                                                                                                               | [references/logs.md](references/logs.md)             |
| `stackbone storage`    | `buckets`, `list --bucket [--prefix --limit --cursor]`, `get <key> --bucket [--out]`, `put <key> --bucket --file`, `presign <key> --bucket`, `remove <key> --bucket`✱                                                                             | [references/storage.md](references/storage.md)       |
| `stackbone rag`        | `collections list/create <name>/remove <name>✱`, `list --collection`, `get <docId> --collection`, `ingest <path> --collection`, `query <text> --collection [--topk]`, `remove <docId> --collection`✱, `jobs`, `retry <jobId>`✱, `cancel <jobId>`✱ | [references/rag.md](references/rag.md)               |
| `stackbone secrets`    | `list`, `set <name> [--value]` (stdin if omitted), `remove <name>`✱ — values are **never** revealed                                                                                                                                               | [references/secrets.md](references/secrets.md)       |
| `stackbone config`     | `get`, `set [--file]` (stdin if omitted), `versions`, `rollback --version`✱, `types` (local codegen, no `--agent`)                                                                                                                                | [references/config.md](references/config.md)         |
| `stackbone contract`   | `show`, `schema`, `capabilities`, `validate`                                                                                                                                                                                                      | [references/contract.md](references/contract.md)     |
| `stackbone hitl`       | `list [--status]`, `get <hitlId>`, `approve <hitlId> [--reason]`✱, `reject <hitlId> [--reason]`✱                                                                                                                                                  | [references/hitl.md](references/hitl.md)             |
| `stackbone prompts`    | `list`, `get <key> [--version]`, `create <key> --name [--template/--file]`, `update <key>`, `remove <key>`✱, `versions <key>`, `rollback <key> --version`✱, `preview <key> [--vars]`                                                              | [references/prompts.md](references/prompts.md)       |
| `stackbone openrouter` | `get`, `models` — read-only; the bearer key value is never returned                                                                                                                                                                               | [references/openrouter.md](references/openrouter.md) |

> `db query` / `db schemas` / `db table` are the same kind of read-only HTTP explorer against the targeted install — they live under `stackbone db` (see [references/db.md](references/db.md)) alongside the drizzle-native migration verbs.

> **No `runs steps` verb.** To trace one run, use `stackbone runs get <runId>` for the header and `stackbone logs tail --run <runId>` for the per-run log lines. (`config types` is the one runtime-group verb that is **local** — it regenerates `.stackbone/config.d.ts` from this project's `config.schema.ts` and takes no `--agent`.)

> **Secrets and OpenRouter plaintext are never revealed by the CLI by design.** `secrets list` only prints masked previews and there is no `secrets get`; `openrouter get` returns mode/public-id/spend-cap but never the bearer value. Reading a plaintext secret is a human-only Studio action behind a re-auth challenge.

---

## Common workflows

Per-command flags, outputs and exit codes live in the `references/` files — these are just the cross-command happy paths.

**Scaffold → iterate → ship:**

```sh
stackbone init my-workspace --with agent # emit the workspace shell + a first eve agent (registers it; needs login)
cd my-workspace && npm install
stackbone add workflow qualify-lead --calls lead-qualifier  # add a durable workflow that delegates to the agent (offline)
stackbone dev                            # server + Studio on 127.0.0.1:4242
# edit agents/<name>/agent/… and workflows/…, the dev session hot-reloads, then:
stackbone link                           # pick org + agent (interactive)
stackbone publish                        # packs dist/stackbone/workspace-bundle.tar
```

> `stackbone init --with empty` (the default) is the fully-offline way to start, then grow the workspace with `stackbone add agent|workflow|workflow-agent`. `add` only writes new files — it never edits your existing TypeScript or `stackbone.config.ts`.

**Database schema:** `stackbone db migrate create <name>` → edit the generated SQL → `stackbone db migrate up`. You never migrate the RAG schema — it's platform-provisioned per install. See [references/db.md](references/db.md).

**Non-interactive (CI / agent shell)** — login is device-flow only (no password, no token env var). A headless run must already have `~/.stackbone/credentials.json` in place (run `stackbone login` once on a machine with a browser, then carry that file into CI). After that, everything else runs with `--json` (and `--yes` for destructive verbs):

```sh
stackbone organization use "$ORG_SLUG" --json   # set the active org (discover slugs with `stackbone list`)
stackbone link --agent "$AGENT_SLUG" --force --json
stackbone db migrate up --json
stackbone publish --json
```

**Discover state before building:** `stackbone metadata --json` (auth + project), `stackbone list --json` (orgs + agents), `stackbone docs sdk` (SDK surface).

## Project link — `.stackbone/project.json`

`init` / `link` write `{ organizationId, agentTemplateId, localDevInstallationId? }` to `.stackbone/project.json`. The `localDevInstallationId` appears once a `stackbone dev` session registers with the control plane; it survives shell restarts and is GC'd after ~7 days idle. **Never commit `.stackbone/`** — `init` gitignores it automatically; if a clone is missing it, re-run `stackbone link` to mint a fresh ID for your machine.
