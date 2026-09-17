---
name: stackbone-cli
description: >-
  Use this skill when driving the `stackbone` CLI to scaffold, run, ship or operate a Stackbone
  workspace: signing in with the device flow (login / logout / whoami), scaffolding a workspace and
  linking it to the organization (init, with an optional first agent / workflow / workflow-agent),
  adding pieces offline (add agent / add workflow / add workflow-agent), running the local emulator
  with Studio (dev), compiling the deployable bundle (build) or writing a deploy folder (package),
  registering a deployed box (link --installation --url --secret --tag), switching the active organization
  (organization use), reading workspace state as JSON (metadata), managing Drizzle migrations
  (db migrate create / up / status), and operating a running installation from the shell:
  discovering targets (agents), listing / inspecting / starting durable workflows (workflows),
  inspecting and controlling runs (runs), tailing logs (logs), browsing the agent DB read-only
  (db query / schemas / table), moving objects (storage), operating the RAG index (rag), setting
  secrets (secrets), versioning the config document (config), deciding approvals (hitl), managing
  the prompt catalog (prompts), inspecting the protocol contract (contract), running eval suites
  (eval), and printing how to reach the docs MCP server (docs).
  Trigger on requests like: scaffold a workspace, add an agent, add a workflow, start the dev
  stack, log in, list my agents, start a workflow by name, retry a failed run, tail the logs, run a
  SELECT against the agent, set a secret, roll the config back, approve an approval, generate a
  migration, build the bundle, package for deploy, register the deployed agent, switch org.
  For writing the code inside the workspace with @stackbone/sdk use the stackbone skill; to triage
  an error or a stuck run use stackbone-debug; to start a new piece from a description use
  stackbone-coder.
license: MIT
metadata:
  author: stackbone
  version: '2.2.0'
  organization: Stackbone
  date: September 2026
---

# Stackbone CLI skill

This skill tells you **how to drive** the `stackbone` CLI: the order of operations, the conventions every command obeys, and the loops you run. It holds no flag tables: every flag, output shape and exit code lives in the docs, one page per command.

## Where the facts live

The docs are served over MCP as the `stackbone-docs` server (`https://docs.stackbone.ai/mcp`, public, tools `search_docs`, `get_doc`, `list_docs`). `stackbone docs` prints the connection details.

- **Start the session with `list_docs` (area `cli`).** It returns every CLI page under its section with a one-line description. [How to find the page](#how-to-find-the-page) below says which page a task needs; take its path from that list and `get_doc` it. Never type a path from memory: pages move, titles stay.
- Before you run a command with a flag you have not verified in this session, read its page.
- When you do not know which command does something, `search_docs` with the question in plain words and `area: "cli"`.
- `stackbone <command> --help` is the local truth for the installed version. When the docs and the binary disagree, the binary wins.

**No `stackbone-docs` tools in your session?** Fetch `https://docs.stackbone.ai/llms.txt`: the same index, one line per page with its title, its description and a link to its raw markdown. Pick the page by title and fetch that link. `https://docs.stackbone.ai/llms-full.txt` is every page in one file (~700 KB).

## How to call the CLI

- The binary is `stackbone` (`pnpm dlx @stackbone/cli <command>` without installing, or a global `pnpm add -g @stackbone/cli`). If `stackbone --help` fails, the CLI is not installed.
- **Pass `--json` on every call** when you are the consumer: one `{ "schema_version": 1, … }` line on stdout, or `{ "error": { "code", "message", "suggestion" } }` on stderr.
- **Pass `--yes` to any destructive verb** (`retry`, `cancel`, `remove`, `rollback`, `approve`, `reject`). Without it the verb exits `5` before any network call.
- **`--json`, `--yes` and `CI=1` all turn prompting off.** A command that would have asked (a value, a picker, a confirmation) errors out instead of hanging, so pass every required value as a flag.
- **Branch on the exit code:** `0` ok, `1` generic, `2` not authenticated (`stackbone login`), `3` no project or `stackbone dev` not running, `4` not found or no registered deployment, `5` permission denied. `error.code` in the envelope is finer (`dev_not_running`, `runtime_not_configured`, …).
- When a command fails, re-run it with `--json` and read `error.suggestion`: it is the next command to try. Then `STACKBONE_LOG_LEVEL=debug` for the CLI's own logs on stderr, then `--verbose` when a `dev` boot stage hangs.

## Session start

Run these before anything else, and act on the exit code:

```sh
stackbone whoami --json     # exit 2 → stackbone login (device flow; --no-browser when headless)
stackbone metadata --json   # auth + linked project + agents in one payload; exit 3 → init or link
```

## The loops

**New workspace.** `stackbone login` → `stackbone init <dir> [--with agent|workflow|workflow-agent] [--name <ws>]` → `cd <dir>` → `stackbone dev`. `init` links the workspace to the org and writes `.stackbone/project.json`, so it needs a signed-in session. It installs the workspace dependencies for you, so do not add an install step of your own; read the command's page when you need to skip or recover that. With a TTY it asks which coding agents to set up (skills + the docs MCP server); non-interactive runs set up none unless you pass `--agents claude-code,cursor,…`.

**Grow the workspace.** `stackbone add agent <name>` / `add workflow <name> [--calls <agent>]` / `add workflow-agent <name>`. Offline, no login, writes only new files, never edits existing TypeScript; a name collision fails unless `--force`. Must run inside a workspace (exit `3` otherwise). Then write the code with the **stackbone** skill.

**Dev loop.** `stackbone dev` boots Postgres + Redis + MinIO, compiles and serves the workspace at `http://127.0.0.1:4242` with hot reload, and opens a tunnel so cloud Studio can reach it. The first run on a fresh machine stops until a model provider is configured (a Studio link is printed; or export `MODEL_PROVIDER_BASE_URL` + `MODEL_PROVIDER_API_KEY` before booting). Then:

```sh
stackbone workflows list --json
stackbone workflows start <name> --input '{"…"}' --json   # validates against the declared schema → { runId }
stackbone runs get <runId> --json
stackbone logs tail --run <runId> --json                   # the per-step frames; --follow for a live run
```

Chat with an agent through Studio's Playground or `POST http://127.0.0.1:4242/openai/v1/chat/completions` with `"model": "<agent-name>"` and any non-empty bearer (`Getting started` (CLI › Guides)).

**Schema change.** Edit `src/schema.ts` → `stackbone db migrate create <name>` → read the generated SQL under `.stackbone/migrations/` → `stackbone db migrate up` → `stackbone db migrate status`. Never edit an applied migration; create a new one. No `BEGIN` / `COMMIT` in migration files.

**Operate an installation.** A verb with no `--agent` targets the local-dev installation and needs `stackbone dev` running (`dev_not_running`, exit `3`, otherwise). `stackbone agents list --json` gives the installation ids; pass `--agent <id>` per call to target a deployed one. Approvals: `stackbone hitl list --status pending --json` → `stackbone hitl approve|reject <id> --yes --json`.

**Ship.** There is no publish command. `stackbone package` writes a deploy folder (compiled workspace + Dockerfile + compose + `.env` + README) someone runs with `docker compose up -d`; `stackbone build` writes only the bundle for an image you own. Then register the running box: `stackbone link --agent <slug> --installation <id> --url <public-url> --secret <hmac-secret> --tag <image-tag>` (the tag is the version; re-run `link` on every deploy). A box answers for one installation, so `--installation` is the id of a `cloud` row of that agent in `stackbone agents list --json`; `link` never creates an installation, so install the agent first. Read `link` (CLI › Reference) before the first `link`: it explains where the secret comes from.

**After it ships, three skills take over.** A box that nobody has wired reaches nothing and measures nothing, and each of these covers one sidebar group the CLI does not reach: **stackbone-integrations** connects a provider, arms a trigger so the agent receives on its own, and sends what it writes somewhere; **stackbone-guardrails** puts limits on a running agent — block a turn, mask data, hold it for a person; **stackbone-evals** turns a manual Playground poke into a suite you can run in CI.

**CI / headless.** Login is device-flow only (no token env var): run `stackbone login` once where a browser exists and carry `~/.stackbone/credentials.json` (chmod 600) into CI. Then `--json` everywhere, `--yes` on destructive verbs, `--agents <list>` on `init` / `link` if you want the coding-agent setup.

## Rules that hold everywhere

- Never commit `.stackbone/` (`init` gitignores it). Run `stackbone dev` in a clone without it and it asks which agent this project is, then writes the file. `link` cannot do that job: it needs a deployed box (`--installation --url --secret --tag`), and with no project file to remember an installation id it refuses.
- Paginated lists take `--limit` + `--cursor` and return `items` + `nextCursor`; pass `nextCursor` back to walk forward. `logs tail` streams instead.
- Secrets are never printed: `secrets list` masks values and there is no reveal verb. Do not try to read one from the CLI.
- `db query` is a single read-only `SELECT`; schema changes go through `db migrate create`. The RAG schema is platform-provisioned; never migrate it by hand.
- `init` takes no `--starter` / `--template`; passing one exits non-zero with a migration message. Per-piece templates live on `add` (`stackbone add workflow <name> --template <t>`).
- Retired commands you may see in old material: `publish`, `openrouter`, `connectors`. They do not exist; use `build` / `package` + `link`, `Gateway` (Home › Features), and Studio's integrations page.
- Interactive prompts protect humans typing `! stackbone …`. When you are the consumer, `--json --yes` is the default.

## How to find the page

- **A flag, a verb, an output shape or an exit code of one command**: the page titled with the command's name under CLI › Reference (`login` / `logout` share `Sign in and out`; `whoami` / `current` / `list` / `organization use` / `metadata` share `Account and organization`).
- **A rule every command obeys** (how a verb picks its target, the JSON envelope, pagination, the `--yes` gate, exit codes): `Command conventions`, and `Configuration reference` for global flags, env vars and on-disk state (both under CLI › Reference).
- **A `dev` boot that stops, or a command that refuses to target**: `Troubleshooting` under CLI › Guides (or the **stackbone-debug** skill).
- **Anything else** (the HTTP wire, shipping, setting up coding agents by hand): `search_docs` with the question in plain words and area `cli`, or area `home` for deployment and setup pages.

## Other skills

- **stackbone**: the code inside the workspace (`defineDeepAgent`, `'use workflow'`, the ambient `stackbone` client).
- **stackbone-debug**: locate the cause of an error, a failed or stuck run, a parked approval.
- **stackbone-coder**: start a new piece from a one-line idea through an interview.
