# `stackbone dev`

Run the agent locally against a fake control plane that mirrors the production contract — auth, LLM gateway, triggers, dry-run billing, persistence. Same contract as production, zero surprises at `stackbone publish` time.

## Synopsis

```sh
stackbone dev [--port <n>] [--studio-port <n>] [--no-docker] [--verbose]
```

## What it boots

| Piece                                   | Default                       | Notes                                                                                                                   |
| --------------------------------------- | ----------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Agent (`/invoke`, `/health`, `/schema`) | `:3000`                       | Hono + Node 24 LTS via `stackbone serve src/index.ts` with hot reload                                                   |
| Stackbone Studio                        | `:4242`                       | Web UI for the local control plane: HITL inbox, runs timeline, db explorer, queues, secrets, config, schema, playground |
| Postgres 17 + `pgvector` + `tsvector`   | `:5432` (docker)              | The agent's Neon stand-in; persisted across restarts of the dev session                                                 |
| MinIO (S3-compatible)                   | `:9002` API / `:9003` console | R2 stand-in; creds `minioadmin / minioadmin`                                                                            |
| Mailpit (SMTP capture)                  | `:1025` SMTP / `:8025` UI     | Catches any outbound email                                                                                              |
| QStash stub (local)                     | inline                        | In-memory queue + scheduler; messages are delivered locally without HMAC verification round-trip                        |

The first run boots `docker compose -f docker/dev/compose.yml up -d` automatically. Subsequent runs reuse the running containers.

## Flags

| Flag                | Description                                                                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--port <n>`        | Agent listen port (default 3000).                                                                                                                             |
| `--studio-port <n>` | Studio listen port (default 4242).                                                                                                                            |
| `--no-docker`       | Skip the docker-compose boot. Use this if you already have Postgres/MinIO/Mailpit running on the standard ports — supply connection strings via `.env.local`. |
| `--verbose`         | Stream every log line. Without this, the dev UI uses a spinner and a tail-tail view of structured logs.                                                       |

## What gets injected

`stackbone dev` simulates the production env-var injection. The agent process sees:

| Variable                                                      | Value in dev                                                                                |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `AGENT_ID`                                                    | `agt_local_dev`                                                                             |
| `AGENT_TEMPLATE_ID`                                           | from `agent.yaml.name`                                                                      |
| `ORGANIZATION_ID`                                             | from `.stackbone/project.json` (or `org_local_dev` if unlinked)                             |
| `STACKBONE_POSTGRES_URL`                                      | `postgres://stackbone:stackbone@localhost:5432/<agent_template>`                            |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `S3_ENDPOINT` | MinIO local creds                                                                           |
| `OPENROUTER_API_KEY`                                          | Your account's dev sub-key (issued at first run, stored in `~/.stackbone/credentials.json`) |
| `QSTASH_TOKEN` / signing keys                                 | Local stub values; receiver runs in-process                                                 |
| `PLATFORM_API_URL`                                            | `http://localhost:4242/api`                                                                 |
| `PLATFORM_API_KEY`                                            | Local stub key                                                                              |

If a key is missing (e.g. you opted out of OpenRouter), `client.ai` calls return `{ data: null, error: { code: 'llm_not_configured', message: '...' } }` — exactly like production with a paused sub-key.

## Hot reload

Saves to anything under `src/` trigger a graceful restart of the agent process. Studio reconnects automatically. Migrations under `migrations/` are **not** auto-applied — run `stackbone db migrate up --all` in a separate shell.

## Triggering an invocation

```sh
# Manual HTTP
curl -X POST http://localhost:3000/invoke \
  -H 'Content-Type: application/json' \
  -d '{"action": "ping"}'

# Or use Studio's Playground at http://localhost:4242 — runs are marked is_playground = true
```

## Logs and traces

- Agent logs stream to the dev shell.
- Studio's Logs tab shows the same logs filtered by `agent_id`, `run_id`, severity.
- OTel traces from the agent are captured by Studio's local Tempo stand-in (visible under the Runs tab).

## Surviving refresh and shell restarts

If you restart your shell, the Postgres/MinIO/Mailpit containers keep running (they're persistent). `stackbone dev` picks up where it left off — no data loss. The `localDevInstallationId` in `.stackbone/project.json` reattaches the session to the control plane so the org's collaborators can see your local install in Studio.

After 7 days of inactivity the control plane GCs the local-dev installation row. To resurrect, run `stackbone dev` again — a new ID is minted automatically.

## Common workflows

### Iterate on `src/index.ts` with the LLM

```sh
stackbone dev --verbose
# Edit src/index.ts — restart on save is automatic
# Open Studio at http://localhost:4242 and use Playground for fast iteration
```

### Reset local state

⚠️ **destructive** — these commands wipe local data:

```sh
# Wipe the agent's local Postgres database
docker compose -f docker/dev/compose.yml down -v
stackbone dev   # boots fresh

# Wipe only MinIO objects, keep Postgres
docker compose -f docker/dev/compose.yml exec minio mc rm -r --force /buckets
```

## Exit codes

| Code | When                                                                                               |
| ---- | -------------------------------------------------------------------------------------------------- |
| 0    | Clean shutdown (`Ctrl-C`)                                                                          |
| 1    | Boot failure (docker not running, port collision, missing `agent.yaml`)                            |
| 3    | Not linked — directory has no `.stackbone/project.json`. Run `stackbone init` or `stackbone link`. |

## Common mistakes

- **Editing `agent.yaml.runtime.entry` while `dev` is running.** The hot-reload watches `src/`; manifest changes need a manual restart.
- **Calling `process.exit()` from the agent.** The wrapper catches it but logs a warning — use `ctx.fail()` or throw to return a proper error envelope.
- **Mounting a non-Hono framework**. The wrapper expects Hono. If you must use Express/Fastify, opt out by setting `runtime.dockerfile: ./Dockerfile` and own the contract — but you lose hot reload, metrics auto-injection, and the embedded Studio.
- **Hitting `:4242` from a fetch inside the agent.** Studio is the control plane in dev; the agent uses `PLATFORM_API_URL = http://localhost:4242/api`, but it's the SDK that calls it, not your code.
