---
name: stackbone-cli
description: >-
  Use this skill when driving the `stackbone` CLI to scaffold, develop, publish or operate an agent on Stackbone:
  authenticating with device flow (login / whoami), creating a new agent project from a starter (init),
  linking an existing directory to an agent template (link), running the local emulator (dev) with Studio,
  serving the runtime contract (serve), publishing to the marketplace (publish), managing database migrations
  with Drizzle (db migrate up / create / status, db studio, db add-rag), reading workspace metadata in JSON
  (metadata), or printing inline docs (docs). Trigger on requests like: scaffold an agent, start the emulator,
  publish my template, generate a migration, list my orgs, see what's linked, log in to Stackbone.
  For writing code inside the agent with @stackbone/sdk, use the stackbone skill instead.
  For triage of errors and stuck runs, use the stackbone-debug skill.
license: MIT
metadata:
  author: stackbone
  version: '0.1.0'
  organization: Stackbone
  date: May 2026
---

# Stackbone CLI

Command-line tool for scaffolding, developing, publishing and operating Stackbone agents.

## Critical: how to call the CLI

The binary is `stackbone` (installed into `node_modules/.bin/` by the starter's `package.json`, exposed by the monorepo on the developer's PATH after `pnpm install`, and globally available if the user installs `@stackbone/cli` themselves). **Always invoke it explicitly**:

```sh
stackbone <command> [--json] [--yes] [--verbose]
```

There is no `npx`-only fallback as of today — the binary is the entrypoint. If `stackbone --help` fails, the user has not run `pnpm install` in a starter or has not installed the global CLI.

**Session start** — verify auth and project:

```sh
stackbone whoami    # who am I, which org
stackbone current   # is this directory linked to an agent template?
```

If not authenticated: `stackbone login` (opens browser; falls back to device-flow code if no browser).
If no project linked: `stackbone init` (new project from a starter) or `stackbone link` (attach to an existing agent template).

## Global options

| Flag           | Description                                                                                                                                                                                    |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--json`       | Emit a versioned JSON envelope (`{ ok, data, error, contract_version }`) instead of human-formatted text. Also **skips value-collection prompts** — errors out if required params are missing. |
| `--yes` / `-y` | Auto-accept Y/N confirmation prompts (deletes, overwrites). Does NOT skip value prompts — combine with `--json` for fully non-interactive runs.                                                |
| `--verbose`    | Stream every log line instead of spinner UI. Useful for CI / agent shells where TTY redraws break parsing.                                                                                     |

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

| Variable                      | Description                                    |
| ----------------------------- | ---------------------------------------------- |
| `STACKBONE_ACCESS_TOKEN`      | Override the stored access token (CI use)      |
| `STACKBONE_ORGANIZATION_ID`   | Override the linked organization               |
| `STACKBONE_AGENT_TEMPLATE_ID` | Override the linked agent template             |
| `STACKBONE_PLATFORM_API_URL`  | Override the control plane URL (dev / staging) |

---

## Commands

### Auth

- `stackbone login` — RFC 8628 device flow. Browser opens; copy the code if it doesn't. See [references/login.md](references/login.md)
- `stackbone logout` — clear stored credentials at `~/.stackbone/credentials.json`
- `stackbone whoami` — show the active user and organization

### Project lifecycle

- `stackbone init [name] --starter <slug>` — scaffold a new agent project from a starter. With no `name` positional, nests the scaffold into a slugified subdirectory of the current dir. See [references/init.md](references/init.md)
- `stackbone link` — attach the current directory to an existing agent template (writes `.stackbone/project.json`)
- `stackbone current` — show the authenticated user and the linked agent template
- `stackbone list` — list organizations the user owns or is a member of, with their templates
- `stackbone metadata` — machine-readable overview of the workspace (auth status, linked project, capabilities, current agent/template ids). **Run this first** with `--json` to discover state before building features.

### Local dev — `stackbone dev`

Run the platform-managed wrapper locally with hot reload and Stackbone Studio at `http://localhost:4242`. The local control plane fakes auth, LLM gateway, triggers, metrics and dry-run billing — same contract as production, zero surprises at publish time. See [references/dev.md](references/dev.md).

```sh
stackbone dev                 # serves the agent at :3000 + Studio at :4242
stackbone dev --port 3010     # custom agent port
stackbone dev --verbose       # stream every log line
```

### Runtime serve — `stackbone serve`

Mount the agent against the canonical contract (`POST /invoke`, `GET /health`, `GET /schema`). This is what the production container runs as its entrypoint; you rarely call it directly, but it's exposed for testing the wrapper in isolation.

```sh
stackbone serve src/index.ts
```

### Publish — `stackbone publish`

Validate `agent.yaml`, run the platform buildpack to package the container, scan with Trivy, sign with cosign, push to `registry.fly.io`, and register a new version of the `agent_template` row. See [references/publish.md](references/publish.md).

```sh
stackbone publish                       # interactive: prompts for version bump
stackbone publish --version 0.3.0 --yes # non-interactive
stackbone publish --dry-run             # build + scan + sign without pushing
```

### Database — `stackbone db`

Manage Drizzle migrations against the agent's dedicated Neon. See [references/db.md](references/db.md).

- `stackbone db migrate create <name>` — generate a new timestamped migration file under `migrations/`
- `stackbone db migrate up [--to <version>] [--all]` — apply pending migrations (advisory lock + journal table; safe to re-run)
- `stackbone db migrate status` — list applied vs pending migrations
- `stackbone db studio` — launch the embedded read-only DB explorer (works against `stackbone dev` or against a cloud `agent` via `--agent <id>`)
- `stackbone db add-rag` — declarative shortcut: generates the schema and indexes the RAG module needs (`_rag_documents`, `_rag_chunks` with `pgvector` and `tsvector`)

> Migrations run inside a backend-managed transaction. Do not put `BEGIN`, `COMMIT` or `ROLLBACK` in your migration files. Filenames are timestamped (`20260518091500_create-users.sql`); the journal lives in the agent's Neon.

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

**Credentials are local-only, chmod 600.** Stored at `~/.stackbone/credentials.json`. Never committed, never shipped in containers. The container at runtime uses `PLATFORM_API_KEY` injected as env var by the platform, not your personal session.

**`.stackbone/project.json` is the link.** Generated by `init` or `link`, contains `{ organizationId, agentTemplateId, localDevInstallationId? }`. Add `.stackbone/` to `.gitignore` — `init` does this automatically.

**`stackbone init` with no positional name nests the scaffold.** A bare `stackbone init --starter ai` slugifies the starter name and creates a subdirectory; `stackbone init my-thing --starter ai` writes into `./my-thing/`. This avoids accidentally polluting the current directory.

**`stackbone dev` boots its own docker-compose.** Postgres 17 with `pgvector` + `tsvector`, MinIO, Mailpit. If you already have these running in another dev session, pass `--no-docker` and supply the connection strings via env vars.

**`stackbone publish` rebuilds even if the source hasn't changed.** The platform buildpack runs against the resolved npm tree, which can change due to lockfile updates. There is no `--cache` flag yet.

**`stackbone db migrate up` is idempotent.** Re-running after partial failure picks up where it left off — the journal table records the last applied version. Never edit a migration file after it has been applied; create a new one instead.

**`stackbone db studio` is read-only by default.** The connection uses a `stackbone_viewer` role with `SET default_transaction_read_only = on` and 5 s `statement_timeout`. To write, use `stackbone db query "<sql>" --writable` (explicit) or run a migration.

**Targeting a cloud agent vs the local-dev install.** Commands that touch runtime data (logs, secrets, db, hitl, runs — these surfaces are still expanding) default to the **local-dev installation** if one is active (`.stackbone/project.json.localDevInstallationId`). To target a cloud `agent`, pass `--agent <id>` per invocation. There is **no** `stackbone use <id>` that persists a target — that hidden state was rejected because it's the exact foot-gun the safety design avoids.

**Tier quota is enforced server-side.** If the org's credit bundle is spent, mutating commands return exit code 5 with `error.code = 'tier_quota_exceeded'` and a `nextActions` block (typically: upgrade tier or wait for next period). Surface this verbatim; do not retry.

---

## Common workflows

### Scaffold a new agent and ship it

```sh
# 1. Pick a starter
stackbone list --starters --json | jq '.data[].slug'

# 2. Scaffold and enter the directory
stackbone init my-agent --starter rag
cd my-agent

# 3. Install deps and start the emulator
pnpm install
stackbone dev

# 4. (in another shell) Run a manual invocation against the emulator
curl -X POST http://localhost:3000/invoke \
  -H 'Content-Type: application/json' \
  -d '{"action": "ingest", "url": "https://example.com/doc.pdf"}'

# 5. Iterate on src/index.ts. When happy, link and publish.
stackbone link             # interactive — pick org + template name
stackbone publish --yes
```

### Set up database schema with migrations

```sh
# Inspect current state
stackbone db migrate status --json

# Create the next migration
stackbone db migrate create create-contracts

# Edit migrations/<timestamp>_create-contracts.sql (CREATE TABLE / ALTER TABLE / CREATE INDEX)

# Apply pending migrations
stackbone db migrate up --all --json
```

> For new RAG collections, prefer `stackbone db add-rag --name contracts` over hand-writing the `pgvector` + `tsvector` schema — it stays in sync with what `client.rag` expects.

### Non-interactive CI / agent shell

```sh
# Authenticate via env var (CI secret)
export STACKBONE_ACCESS_TOKEN=...

# Or login with email+password if your CI provider supports interactive (rare)
# stackbone login --email "$EMAIL" --password "$PASSWORD" --json

stackbone link --org-id "$ORG_ID" --agent-template-id "$TEMPLATE_ID" --yes --json
stackbone db migrate up --all --json
stackbone publish --version "$VERSION" --yes --json
```

### Discover what's available before building

```sh
# Auth + project state
stackbone metadata --json

# Available starters (also reachable via the marketplace)
stackbone list --starters --json

# SDK surface
stackbone docs sdk
```

---

## Project configuration

After `init` or `link`, `.stackbone/project.json` is created in the project root:

```json
{
  "organizationId": "org_01HX...",
  "agentTemplateId": "tpl_01HX...",
  "localDevInstallationId": "ins_01HX..."
}
```

The `localDevInstallationId` is present when a `stackbone dev` session is registered with the control plane (Feature 42, first-class local-dev installations). It survives shell restarts and gets GC'd after 7 days of inactivity.

> **Never commit `.stackbone/`** — `stackbone init` adds it to `.gitignore` automatically. If you cloned a repo where someone forgot, re-add it and run `stackbone link` to mint a fresh ID for your machine.
