# `stackbone dev`

Boot the whole workspace locally against a local control plane that mirrors the production contract — auth, LLM gateway, triggers, persistence. Same contract as production, zero surprises at `stackbone publish` time. Your deep agents run **in-process** behind a single HTTP server (with Stackbone Studio mounted) that listens on **`127.0.0.1:4242`** by default.

## Synopsis

```sh
stackbone dev [--port <n>] [--listen] [--auto-migrate|--no-auto-migrate] [--verbose]
stackbone dev --print-contract      # print the advertised JSON contract and exit (no boot)
```

## What it boots

| Piece                 | Default          | Notes                                                                                                                                                                       |
| --------------------- | ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Server + Studio       | `127.0.0.1:4242` | One HTTP server with Studio mounted: HITL inbox, runs timeline, db explorer, workflows, secrets, config, schema, playground. `--listen` binds `0.0.0.0`; `--port` moves it. |
| Deep agents           | in-process       | Each `deep-agents/<name>/` is bundled once, its LangGraph graph built and cached **in the server process** — no subprocess, no port — and hot-swapped on save.              |
| Workflows             | in-process       | Built once at boot, then served behind the server's `/api/workflows/*` routes.                                                                                              |
| Postgres              | docker           | The agent's database; persisted across restarts of the dev session.                                                                                                         |
| Redis                 | docker           | Backs the durable workflow execution (the Workflow SDK World) — not a stub.                                                                                                 |
| MinIO (S3-compatible) | docker           | Object-store stand-in for `stackbone.storage`.                                                                                                                              |
| Tunnel                | on               | A dev tunnel exposes the session to the control plane (your org sees the local install in Studio); the JSON envelope carries the `public_url`.                              |

The data services come up automatically on the first run; later runs reuse the running containers.

## Flags

| Flag                                   | Description                                                                                                                |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `--port <n>`                           | Server + Studio port (default 4242).                                                                                       |
| `--listen`                             | Bind the HTTP server to `0.0.0.0` (reachable from your LAN). Default binds to `127.0.0.1` only.                            |
| `--auto-migrate` / `--no-auto-migrate` | Force the schema watcher's auto-migrate on/off for this session. Unset = `agent.yaml`'s `dev.autoMigrate`, default **on**. |
| `--verbose`                            | Stream every log line instead of the per-stage spinner UI. Useful in CI / agent shells.                                    |
| `--print-contract`                     | Print the JSON contract this CLI advertises and exit without booting. Works with no project linked.                        |

> There is **no** `--studio-port` (Studio shares the one `:4242` server) and **no** `--no-docker` — the local stack always boots its own data services.

## What gets injected

`stackbone dev` simulates the production env-var injection. The workspace process sees the same env contract the runtime injects in cloud / self-host:

| Variable                                     | Value in dev                                                                                       |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `HMAC_SECRET`                                | Per-install secret used to sign the runtime's brokered calls (connectors, workflows).              |
| `STACKBONE_INSTALLATION_ID`                  | The local-dev installation id.                                                                     |
| `STACKBONE_SECRET_KEY`                       | Per-agent key that decrypts `stackbone.secrets`.                                                   |
| `DATABASE_URL`                               | The agent's local Postgres (also exported as `STACKBONE_POSTGRES_URL` for the db migration verbs). |
| `AGENT_ID` / `WORKSPACE_ID`                  | Identity of the agent + workspace.                                                                 |
| `WORKFLOW_REDIS_URL`                         | The local Redis backing durable workflow execution.                                                |
| `OPENROUTER_API_KEY` / `OPENROUTER_BASE_URL` | Your org's dev OpenRouter wiring (seeded from the control plane at first run).                     |

If a key is missing (e.g. you opted out of OpenRouter), the relevant `stackbone.*` call surfaces the error in its `Result` envelope — exactly like production with a paused sub-key.

## Hot reload

- **Deep agents** — saving anything under `deep-agents/<name>/` re-bundles **only that agent** and hot-swaps its graph in the registry, typically well under 2s; the process never restarts and an in-flight turn finishes on the old graph. A broken save paints a red "fix the error and save again" status while the previous graph keeps serving.
- **Workflows** — saving a `workflows/` file rebuilds and remounts the workflow set.
- **`config.schema.ts`** — regenerates `.stackbone/config.d.ts`.
- **`src/schema.ts` / new migrations** — the schema watcher applies pending migrations automatically when auto-migrate is on (the default; disable with `--no-auto-migrate`). Deep agents read the migrated Postgres live — no relaunch.

## Generated types

While `stackbone dev` runs, it (re)writes two git-ignored type files: `.stackbone/agents.d.ts` (the typed agent-name union `callDeepAgent` narrows against, refreshed on every deep-agent reload) and `.stackbone/connect.d.ts` (types for the connections you address with `stackbone.connection('<id>')`, derived from each connector's action schema).

## Reaching the running workspace

Deep agents speak the **standard wire** — any OpenAI/Anthropic client works by pasting a base URL; the `model` field selects the agent. In dev, any non-empty bearer is accepted.

```sh
# Chat with an agent (Anthropic Messages; agent = the `model` field):
curl -N -X POST http://127.0.0.1:4242/anthropic/v1/messages \
  -H 'authorization: Bearer stackbone-dev' \
  -H 'content-type: application/json' \
  -d '{"model":"<name>","max_tokens":1024,"stream":true,"messages":[{"role":"user","content":"hello"}]}'

# Same agent over OpenAI Chat Completions:
curl -N -X POST http://127.0.0.1:4242/openai/v1/chat/completions \
  -H 'authorization: Bearer stackbone-dev' \
  -H 'content-type: application/json' \
  -d '{"model":"<name>","stream":true,"messages":[{"role":"user","content":"hello"}]}'

# Durable server-side session: add the header and send ONLY the new message each turn —
# the server rehydrates history (and it survives a `stackbone dev` restart):
#   -H 'x-stackbone-session: my-session-1'
# No header ⇒ stateless: the client replays the full history itself.

# List the agents (populates client model dropdowns):
curl http://127.0.0.1:4242/openai/v1/models -H 'authorization: Bearer stackbone-dev'

# Start a workflow:
curl -X POST http://127.0.0.1:4242/api/workflows/<name>/start \
  -H 'content-type: application/json' \
  -d '{"orderId":"o_1"}'

# Or drive the agent from Studio's playground at http://127.0.0.1:4242 — UI-initiated runs are marked is_playground = true.
```

## Surviving refresh and shell restarts

If you restart your shell, the Postgres / Redis / MinIO containers keep running. `stackbone dev` picks up where it left off — no data loss (durable sessions included: the checkpointer state lives in Postgres). The `localDevInstallationId` in `.stackbone/project.json` reattaches the session to the control plane so the org's collaborators can see your local install in Studio. After ~7 days of inactivity the control plane GCs the local-dev installation row; the next `stackbone dev` mints a new id.

## Exit codes

| Code | When                                                                                               |
| ---- | -------------------------------------------------------------------------------------------------- |
| 0    | Clean shutdown (`Ctrl-C`)                                                                          |
| 1    | Boot failure (docker not running, port collision, bad `--port`)                                    |
| 3    | Not linked — directory has no `.stackbone/project.json`. Run `stackbone init` or `stackbone link`. |

## Common mistakes

- **Chatting via a proprietary route.** There is no `/api/agents/:name/chat` — agents are served over the standard `/anthropic/v1/messages` + `/openai/v1/chat/completions` endpoints, selected by `model`.
- **Sending `x-stackbone-session` AND replaying history.** With the session header the server rehydrates history — send only the new message, or you duplicate context.
- **Reaching `:4242` from inside the agent.** That server is the control plane in dev; the SDK already talks to it for you via the injected env — your agent code never hard-codes that URL.
- **Adding the FIRST deep agent while dev runs.** The `deep-agents/` watcher arms at boot only if the folder exists — a workspace that gained its first agent mid-session needs one dev restart.
