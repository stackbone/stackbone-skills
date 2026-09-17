---
name: stackbone-debug
description: >-
  Use this skill when triaging an error, a failure or unexpected behavior in a Stackbone
  workspace: a deep agent or a durable workflow running locally under `stackbone dev`, or an
  installation deployed in the customer's cloud. Covers SDK errors (the { data, error } envelope
  and its `<prefix>_<reason>` codes), thrown errors from stackbone.database, callDeepAgent and
  stackbone.connection(id), HTTP 4xx/5xx from the control plane or the box, CLI error codes
  (dev_not_running, runtime_not_configured, …), a `stackbone dev` boot that stops or fails,
  durable run failures and timeouts, a workflow start rejected on its input schema
  (workflow_input_invalid), HITL stuck at either level (a workflow parked on requestApproval(),
  an agent turn interrupted on a gated tool), durable-session issues (memory not surviving),
  database slow queries / pgvector mismatches, secrets decryption errors, connector call failures
  (no installation, a bad credential), build failures (native dep, a second SDK copy), and tier / billing
  402 responses. Trigger on requests like: my agent isn't working, why is this 500, this run hung,
  the inbox is empty but the run is paused, the agent forgot the conversation, the workflow
  rejected my input, dev won't boot, why am I getting 402, my connector call is rejected, the
  model returns nothing. This skill locates the cause and does not propose fixes: once found,
  switch to the stackbone skill (code) or the stackbone-cli skill (commands) for the fix.
license: MIT
metadata:
  author: stackbone
  version: '2.2.0'
  organization: Stackbone
  date: September 2026
---

# Stackbone debug skill

This skill tells you **how to locate** the cause of a failure in a Stackbone workspace. It holds no error catalog: the meaning of each code, status and message lives in the docs, and you read it from there once you have the code in hand.

## Where the facts live

The docs are served over MCP as the `stackbone-docs` server (`https://docs.stackbone.ai/mcp`, public, tools `search_docs`, `get_doc`, `list_docs`). `stackbone docs` prints the connection details.

- **Start the session with `list_docs`.** It returns every page under its area and section. [How to find the owner page](#how-to-find-the-owner-page) below says which page a code or a symptom belongs to; take its path from that list and `get_doc` it. Never type a path from memory: pages move, titles stay.
- Once you hold an error code, read the page that owns it: its error section says what the code means and what to check next.
- Once you hold a message with no code, `search_docs` with the message verbatim, then with the symptom in plain words.
- Do not explain a code from memory. Codes are stable but their causes and the next command change between releases.

**No `stackbone-docs` tools in your session?** Fetch `https://docs.stackbone.ai/llms.txt`: the same index, one line per page with its title, its description and a link to its raw markdown. Pick the page by title and fetch that link. `https://docs.stackbone.ai/llms-full.txt` is every page in one file (~700 KB).

## How to use this skill

1. **Capture the symptom verbatim**: the error message, the HTTP status, the run id, the exact command. Never paraphrase it.
2. **Get a code.** Re-run the CLI command with `--json` and read `error.code` + `error.suggestion`; read `result.error.code` from an SDK call; read the status and body of an HTTP response; read the run's log frames.
3. **Run the first diagnostic** from the tables below, as-is. Every command here is read-only. A mutation (`retry`, `cancel`, `approve`, `reject`) is marked ⚠️ and must be proposed to the user before it runs.
4. **Read the owner page** ([how to find it](#how-to-find-the-owner-page)) for what the code means and what to check next.
5. **Report what you found** with the exact command output, then hand off: a code change goes to the **stackbone** skill, a command or a migration to the **stackbone-cli** skill.

## Locate a failing run

Every agent turn and every workflow run is a durable run with a status (`running` / `done` / `failed` / `interrupted`) and a log. The local-dev installation is the default target and needs `stackbone dev` running; pass `--agent <id>` (from `stackbone agents list --json`) for a deployed one.

```sh
stackbone runs list --status failed --json          # find the run id
stackbone runs get <runId> --json                   # header: status, trigger, timing
stackbone logs tail --run <runId> --json            # the per-step frames: each 'use step', its retries, the error that failed it
stackbone logs tail --level error --json --follow   # watch a live installation
```

There is no `runs steps` verb: the step waterfall is in the log frames. Studio's Runs tab shows the same timeline.

## Symptom → first diagnostic

| Symptom                                                                 | First diagnostic                                                                                                                                                                                                                |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A `stackbone` command fails                                             | Re-run with `--json`; read `error.code` + `error.suggestion`; escalate with `STACKBONE_LOG_LEVEL=debug`, then `--verbose` for a boot stage                                                                                      |
| Exit `3` with no project / `dev_not_running`                            | `stackbone metadata --json`; is `.stackbone/project.json` there, is `stackbone dev` up                                                                                                                                          |
| Exit `4` `runtime_not_configured` against `--agent`                     | `stackbone agents get <slug> --json`: the box has no registered deployment                                                                                                                                                      |
| `stackbone dev` stops or nothing of yours starts                        | Read the boot banner; the first run waits for a model provider (a Studio link is printed)                                                                                                                                       |
| Workflow start rejected, no run created (HTTP 400)                      | `stackbone workflows schema <name> --json`; compare the body to `issues[]` in the 400                                                                                                                                           |
| `workflow_not_found` / `workflows list` shows nothing                   | Is there a `workflows/<name>.workflow.ts`; is `dev` serving it                                                                                                                                                                  |
| A step retries then the run fails                                       | `stackbone logs tail --run <runId> --json`: the recurring `error.cause`; is the step idempotent                                                                                                                                 |
| Run stuck `running` with no progress                                    | It is parked on a `requestApproval()` gate: `stackbone hitl list --status pending --json`                                                                                                                                       |
| Run paused but the inbox is empty                                       | `stackbone logs tail --run <runId> --json` for `approval_persist_failed`; or the pause timed out and the `fallback` fired                                                                                                       |
| Agent turn `interrupted` on a gated tool; new messages rejected         | Expected until decided: `stackbone hitl list --status pending --json` → ⚠️ `stackbone hitl approve <id> --yes --json`                                                                                                           |
| The agent forgot the conversation                                       | Did the client send `x-stackbone-session`; does the workspace have the checkpointer peer (the boot log warns once)                                                                                                              |
| Run succeeds with no output                                             | The body returned `undefined` instead of an object matching `outputSchema`                                                                                                                                                      |
| A schedule never fires or fires twice                                   | `stackbone workflows list --json`; `stackbone.workflows.listSchedules()` from a step                                                                                                                                            |
| `stackbone.connection(id)` throws                                       | Read `err.code`: `connector_installation_required` → the operator has not installed that connector in Studio; `credential_error` / `no_token` → its credential. The id is the connector's verbatim id, not a per-connection one |
| Storage 404 on a key just uploaded; signed URL 403                      | Was the upload's `error` checked; did the URL TTL elapse; is local MinIO up                                                                                                                                                     |
| Slow query, wrong `pgvector` order, empty `tsvector` search             | `stackbone db schemas --json`, `stackbone db query "<single SELECT>" --json`; `EXPLAIN` needs `psql` on `STACKBONE_POSTGRES_URL`                                                                                                |
| `stackbone build` / `package` aborts                                    | The message names the native dep or the second SDK copy (a nested `node_modules` or a per-agent pin)                                                                                                                            |
| `stackbone.secrets.get` returns `secrets_not_found` / `_decrypt_failed` | `stackbone secrets list --json` (names only, values masked)                                                                                                                                                                     |
| A model call returns nothing, 429, 402                                  | `result.error.code` (`ai_*`); check the model provider on Studio's gateway screen                                                                                                                                               |

## How to find the owner page

- **An SDK code** (`<prefix>_<reason>` from `result.error.code`): `search_docs` the code verbatim with area `sdk`. The owner page lists it in its error table, so the top hit is the page. The prefix names the surface (`ai_*` → `stackbone.ai`, `rag_*` → `stackbone.rag`, `secrets_*` → `stackbone.secrets`, …); two do not: `s3_*` belongs to `stackbone.storage` and `approval_*` to `Human-in-the-loop`. `contract_*` and `capability_unavailable` are explained on `@stackbone/sdk overview` (SDK › Reference).
- **A `ConnectorCallError` code** (`connector_installation_required`, `credential_error`, `invalid_args`, …): `stackbone.connection` (SDK › Platform), which carries the full list. Match `err.code`, never `instanceof`.
- **A CLI `error.code`** (`dev_not_running`, `runtime_not_configured`, `no_project`, …) or an exit code: `Troubleshooting` (CLI › Guides), then `Configuration reference` (CLI › Reference) for the exit-code table.
- **An HTTP status from the box or the control plane**: `search_docs` the status and the route (`400 workflow_input_invalid`, `404 workflow_not_found`); the workflow routes live under CLI › Agent protocol. A 402 body (`tier_quota_exceeded`, `*_cap_reached`) is not an `SdkError` and is not retryable: surface its message verbatim; the organization owner upgrades.
- **A run you want to read after the fact**: `Logging & observability` (SDK › Platform); a box that will not register or answer: `Troubleshooting` under Home › Deploy.

## Three things to keep straight

- **Three surfaces throw, the rest return `{ data, error }`.** `stackbone.database` throws Drizzle/Postgres errors; `callDeepAgent` / `streamDeepAgent` throw and fail the enclosing step; `stackbone.connection(id)` throws a `ConnectorCallError` (match `err.code`, never `instanceof`). Everything else puts the code in `result.error.code`, with `meta` and `cause` as the structured escape hatches.
- **A 402 is a control-plane body, not an SDK code.** The SDK-side twin of a spent credit bundle is `ai_credits_exhausted`.
- **A 404 on `--agent <id>` from another organization is expected.** Studio uses the same 404 to avoid leaking existence.

## What this skill does not cover

Runtime performance tuning (cold start, sizing), cost analysis beyond the 402 bodies, and custom OTel pipelines. Studio's Logs and Costs tabs are the supported surfaces for those today.

## Other skills

- **stackbone**: the fix, when it is code (`defineDeepAgent`, a step, an ambient call).
- **stackbone-cli**: the fix, when it is a command, a migration or a deploy step.
- **stackbone-coder**: start a new piece from a one-line idea through an interview.
