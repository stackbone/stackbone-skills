# `stackbone dev`

Boot the whole workspace locally against a local control plane that mirrors the production contract — auth, LLM gateway, triggers, persistence. Same contract as production, zero surprises at `stackbone publish` time. Your durable [eve](https://eve.dev/docs/introduction) agents run as subprocesses behind a single HTTP server (with Stackbone Studio mounted) that listens on **`127.0.0.1:4242`** by default.

## Synopsis

```sh
stackbone dev [--port <n>] [--listen] [--verbose]
stackbone dev --print-contract      # print the advertised JSON contract and exit (no boot)
```

## What it boots

| Piece                 | Default          | Notes                                                                                                                                                                                  |
| --------------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Server + Studio       | `127.0.0.1:4242` | One HTTP server with Studio mounted: HITL inbox, runs timeline, db explorer, workflows, secrets, config, schema, agent chat (Sessions). `--listen` binds `0.0.0.0`; `--port` moves it. |
| Agents                | free ports       | Each eve agent in the workspace boots as a subprocess on a free port behind the server, with hot reload.                                                                               |
| Workflows             | in-process       | Built once at boot, then served behind the server's `/api/workflows/*` routes.                                                                                                         |
| Postgres              | docker           | The agent's database; persisted across restarts of the dev session.                                                                                                                    |
| Redis                 | docker           | Backs the durable workflow execution (the Workflow SDK World) — not a stub.                                                                                                            |
| MinIO (S3-compatible) | docker           | Object-store stand-in for `stackbone.storage`.                                                                                                                                         |

The data services come up automatically on the first run; later runs reuse the running containers.

## Flags

| Flag               | Description                                                                                         |
| ------------------ | --------------------------------------------------------------------------------------------------- |
| `--port <n>`       | Server + Studio port (default 4242).                                                                |
| `--listen`         | Bind the HTTP server to `0.0.0.0` (reachable from your LAN). Default binds to `127.0.0.1` only.     |
| `--verbose`        | Stream every log line instead of the per-stage spinner UI. Useful in CI / agent shells.             |
| `--print-contract` | Print the JSON contract this CLI advertises and exit without booting. Works with no project linked. |

> There is **no** `--studio-port` (Studio shares the one `:4242` server) and **no** `--no-docker` — the local stack always boots its own data services.

## What gets injected

`stackbone dev` simulates the production env-var injection. The agent process sees the same env contract the runtime injects in cloud / self-host:

| Variable                                     | Value in dev                                                                                       |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `HMAC_SECRET`                                | Per-install secret used to sign/verify the agent's HTTP requests.                                  |
| `STACKBONE_INSTALLATION_ID`                  | The local-dev installation id.                                                                     |
| `STACKBONE_SECRET_KEY`                       | Per-agent key that decrypts `stackbone.secrets`.                                                   |
| `DATABASE_URL`                               | The agent's local Postgres (also exported as `STACKBONE_POSTGRES_URL` for the db migration verbs). |
| `AGENT_ID` / `WORKSPACE_ID`                  | Identity of the agent + workspace.                                                                 |
| `WORKFLOW_REDIS_URL`                         | The local Redis backing durable workflow execution.                                                |
| `AGENT_URLS`                                 | The map of sibling-agent name → local URL, so one agent can call another.                          |
| `OPENROUTER_API_KEY` / `OPENROUTER_BASE_URL` | Your org's dev OpenRouter wiring (seeded from the control plane at first run).                     |

If a key is missing (e.g. you opted out of OpenRouter), the relevant `stackbone.*` call surfaces the error in its `Result` envelope — exactly like production with a paused sub-key.

## Hot reload

Saves under your agent / workflow source trigger a graceful restart of the affected agent process; Studio reconnects automatically. Editing `config.schema.ts` regenerates `.stackbone/config.d.ts`. Database migrations are **not** auto-applied — run `stackbone db migrate up` in a separate shell.

## Generated connector types

While `stackbone dev` runs, it (re)writes `.stackbone/connect.d.ts` — a git-ignored type file (like the Drizzle migrations output) that types the connections you address with `stackbone.connection('<id>')`, derived from each connector's action schema. The runtime call stays generic, so a new community connector needs no SDK release. See the **stackbone** skill's connect doc for how to reference the type in your code.

## Reaching the running workspace

```sh
# Chat with an agent through the server's front door (the caller does NOT sign):
curl -N -X POST http://127.0.0.1:4242/api/agents/<name>/chat \
  -H 'content-type: application/json' \
  -d '{"message":"hello"}'

# Start a workflow:
curl -X POST http://127.0.0.1:4242/api/workflows/<name>/start \
  -H 'content-type: application/json' \
  -d '{"orderId":"o_1"}'

# Or drive the agent from Studio's chat UI at http://127.0.0.1:4242 — UI-initiated runs are marked is_playground = true.
```

## Surviving refresh and shell restarts

If you restart your shell, the Postgres / Redis / MinIO containers keep running. `stackbone dev` picks up where it left off — no data loss. The `localDevInstallationId` in `.stackbone/project.json` reattaches the session to the control plane so the org's collaborators can see your local install in Studio. After ~7 days of inactivity the control plane GCs the local-dev installation row; the next `stackbone dev` mints a new id.

## Exit codes

| Code | When                                                                                               |
| ---- | -------------------------------------------------------------------------------------------------- |
| 0    | Clean shutdown (`Ctrl-C`)                                                                          |
| 1    | Boot failure (docker not running, port collision, bad `--port`)                                    |
| 3    | Not linked — directory has no `.stackbone/project.json`. Run `stackbone init` or `stackbone link`. |

## Common mistakes

- **Expecting an agent on `:3000`.** There is no separate agent port — everything is reached through the `:4242` server's `/api/agents/*` and `/api/workflows/*` routes (the agents themselves bind free ports internally).
- **Reaching `:4242` from inside the agent.** That server is the control plane in dev; the SDK already talks to it for you via the injected env — your agent code never hard-codes that URL.
- **Forgetting to apply migrations.** Hot reload restarts the agent on source saves, but pending migrations are applied only when you run `stackbone db migrate up`.
