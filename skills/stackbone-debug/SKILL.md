---
name: stackbone-debug
description: >-
  Use this skill when triaging an error, failure or unexpected behavior in a Stackbone agent —
  either running locally under `stackbone dev` or deployed in production. Covers SDK errors
  ({ data, error } envelope codes), HTTP 4xx/5xx from the control plane, run failures and timeouts,
  HITL runs stuck waiting for approval, database slow queries / missing indexes / pgvector mismatches,
  secrets decryption errors, queue (QStash) signature verification failures, deploy failures
  (Trivy CVE block, cosign signing, registry push), tier quota exceeded responses, and OpenRouter
  rate-limit / billing-paused situations. Trigger on requests like: my agent isn't working,
  why is this 500, this run hung, the HITL inbox is empty but the run is paused, publish failed,
  why am I getting 402, the LLM call returns no response. The skill guides diagnostic command
  execution; it does not propose fixes — once you've located the cause, switch back to the stackbone
  or stackbone-cli skill to implement the fix.
license: MIT
metadata:
  author: stackbone
  version: '0.1.0'
  organization: Stackbone
  date: May 2026
---

# Stackbone debug & diagnostics

This skill helps an AI coding agent **locate** the source of a failure in a Stackbone agent — it does not propose fixes. Once the cause is identified, switch back to the **stackbone** skill (for SDK / `agent.yaml` changes) or the **stackbone-cli** skill (for CLI / db / publish actions) to implement the fix.

## How to use this skill

1. Capture the symptom verbatim from the user (error message, HTTP status, stuck behavior, unexpected output).
2. Match the symptom to a row in the tables below.
3. Run the diagnostic command **as-is** (these are read-only, safe).
4. Report what you found in plain narrative — exact command output, not paraphrase.
5. Hand off to `stackbone` or `stackbone-cli` for the fix.

> **Read-only by design.** No command in this skill mutates state. If a diagnostic requires a mutation (e.g. resetting a stuck job), it's flagged with **⚠️ destructive** and the agent must surface the proposal to the user before executing.

---

## SDK errors — `{ data, error }` envelope

Every `client.*` method returns `{ data, error }`. The `error` is `null` on success; otherwise it carries `{ code, message, details, retryable }`.

| `error.code`                 | Where it comes from                                                                  | First diagnostic                                                                                    |
| ---------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| `not_authenticated`          | `PLATFORM_API_KEY` missing or invalid                                                | `stackbone metadata --json` (check auth status)                                                     |
| `tier_quota_exceeded`        | Org's credit bundle is spent                                                         | Surface `error.nextActions` verbatim — do not retry                                                 |
| `capability_not_granted`     | Trying to use a module (e.g. `client.rag`) not declared in `agent.yaml.capabilities` | Read `agent.yaml`; if the capability is missing, that's the fix                                     |
| `validation_failed`          | Input did not match `defineAgent({ schema })`                                        | `error.details` has the Zod-style path + reason — inspect verbatim                                  |
| `db_constraint_violation`    | Drizzle / Postgres FK/unique/check failure                                           | `error.details.constraint` names the violated constraint; `stackbone db migrate status` for context |
| `db_connection_error`        | Neon hibernated or transient                                                         | Wait ~3 s and retry once; if persistent, check Neon Console                                         |
| `rag_ingest_failed`          | LlamaParse parse error, file too big, unsupported format                             | `error.details.file` + `error.details.reason`; check `rag.parser` in `agent.yaml`                   |
| `llm_rate_limited`           | OpenRouter throttling                                                                | `error.details.retry_after_ms` — back off and retry                                                 |
| `llm_billing_paused`         | Stackbone sub-key disabled (org credits spent)                                       | Same as `tier_quota_exceeded` — surface and stop                                                    |
| `storage_not_found`          | Object key doesn't exist in the bucket                                               | Check the `key` you saved vs what's in `_storage_objects`                                           |
| `storage_signed_url_expired` | Pre-signed URL TTL elapsed                                                           | Regenerate with `client.storage.signedUrl()`                                                        |
| `queue_signature_invalid`    | QStash receiver HMAC check failed                                                    | Check `QSTASH_CURRENT_SIGNING_KEY` vs `QSTASH_NEXT_SIGNING_KEY` rotation state                      |
| `approval_already_decided`   | `client.approval.resume()` on an approval that is already approved/rejected          | `client.approval.get(id)` shows the decision                                                        |
| `connection_not_authorized`  | `client.connections.from('notion').client()` but the org never authorized Notion     | Check `connections:` in `agent.yaml`; the org must complete OAuth in Studio                         |

---

## HTTP errors from the control plane

When you call the platform API directly (`PLATFORM_API_URL`), or when `stackbone <command>` reports a backend error:

| Status | Meaning                                                                                              | Diagnostic                                                                                                          |
| ------ | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| 401    | Not authenticated                                                                                    | `stackbone login` (interactive) or check `STACKBONE_ACCESS_TOKEN` (CI)                                              |
| 402    | `tier_quota_exceeded` — org credit bundle spent                                                      | Read `body.nextActions`; org owner must upgrade                                                                     |
| 403    | RBAC capability mismatch (e.g. `approver` role doing an `owner`-only action)                         | Body's `capability` field tells you which one is needed                                                             |
| 404    | Resource not found OR cross-org leak prevention                                                      | If targeting `--agent <id>` from another org, that's expected — Studio uses the same 404 to avoid leaking existence |
| 409    | State conflict (publishing a version that already exists, linking a directory that's already linked) | Read the body's `conflict` field                                                                                    |
| 422    | Validation failed                                                                                    | Body's `errors[]` array lists path + reason                                                                         |
| 5xx    | Backend issue                                                                                        | `stackbone logs platform --limit 50` (if you have access) and retry once with exponential backoff                   |

---

## Runs failing or timing out

In `stackbone dev` Studio at `http://localhost:4242` or in production Studio:

```sh
# (CLI surface for runs is rolling out — see ADR 2026-05-15 for the runtime surface roadmap)
# In the meantime, inspect via Studio's Runs tab — sort by failed, click into the timeline
```

Common causes:

| Symptom                                        | Likely cause                                                                                     | Diagnostic                                                                              |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------- |
| Run timeout at 30 s exactly                    | Non-streamed LLM completion exceeded the invocation budget                                       | Check `src/index.ts` for `stream: false` on long completions — switch to `stream: true` |
| Run timeout at the platform's max              | Synchronous loop over many items                                                                 | Move the loop to `client.queues.publish()` for async fan-out                            |
| Run fails immediately with `validation_failed` | Schema mismatch between `input` and `defineAgent({ schema })`                                    | The run timeline has `validation.errors[]`                                              |
| Run "paused" forever with no inbox entry       | `client.approval.requestAndWait` called but the request failed silently in a `try { } catch { }` | Search the code for `requestAndWait` — destructure `{ data, error }` and check          |
| Run succeeds but no output                     | `ctx.ok()` not called or called with `undefined`                                                 | Add `client.observability.log('debug', 'before ctx.ok', { keys: Object.keys(result) })` |

---

## HITL runs stuck

When the org member says "the agent is paused but nothing is in the inbox":

1. The approval likely **failed to create** because the agent's `agent.yaml` doesn't declare `capabilities: [hitl]`. Check `agent.yaml`.
2. Or the approval was created but **rejected on the approver side and never resumed** by the agent. Check Studio's HITL tab for resolved-but-stale entries.
3. Or the **approver role isn't assigned** to any org member. `stackbone metadata --json` shows the org's roles.

---

## Database — slow queries, missing indexes, pgvector

```sh
# (db diagnostic CLI surface is rolling out — for now use db studio + Postgres standard tooling)
stackbone db studio
```

In `db studio`:

- **Slow `SELECT`** — run `EXPLAIN (ANALYZE, BUFFERS) <query>` in the SQL console; look for sequential scans on large tables.
- **`pgvector` distance returns wrong order** — confirm the index uses the **same distance operator** as your query (`<->` cosine, `<#>` inner product, `<=>` L2). A query with `<->` against an index built for `<#>` does a sequential scan and returns wrong results.
- **`tsvector` queries return nothing** — confirm the column was populated (`UPDATE … SET search_tsv = to_tsvector(…)` on existing rows; the trigger only fires on writes after creation).
- **Migration applied but the table isn't there** — `stackbone db migrate status` will show `applied` for a migration that failed mid-transaction in old SDK versions. Read the journal table directly: `SELECT * FROM _migrations ORDER BY version DESC LIMIT 5`.

---

## Storage — missing files, signed URLs failing

| Symptom                                                   | Diagnostic                                                                                         |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| 404 on a key you just uploaded                            | The upload returned `error` and you didn't check; or you saved `key` to DB before `await` resolved |
| Signed URL returns 403 after 5 min                        | The URL TTL elapsed — `signedUrl(key, { expiresIn: 3600 })`                                        |
| Object exists in R2 console but not in `_storage_objects` | Someone uploaded directly to R2, bypassing the SDK — `client.storage.list()` won't show it         |
| Local dev: uploads silently disappear                     | MinIO container not running — `docker compose ps`                                                  |

---

## Queues — QStash failures

| Symptom                                      | Diagnostic                                                                                                                                            |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Receiver returns 401 to QStash               | HMAC verification failed — confirm `QSTASH_CURRENT_SIGNING_KEY` is the **current** key and `QSTASH_NEXT_SIGNING_KEY` is the **next** (rotation state) |
| Message published but receiver never invoked | Check Studio's Queues tab for dead-letter (after configured max retries the message goes to DLQ); inspect `error.lastAttemptedAt` and `error.message` |
| Scheduled job fires twice                    | The agent has two replicas, and the schedule isn't using the QStash scheduler — use `client.queues.schedule()`, not a setInterval inside the agent    |

---

## Deployments — `stackbone publish` failed

| Stage           | Failure                     | Diagnostic                                                                                                                    |
| --------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Validation      | `agent.yaml` schema invalid | The error has the field path; cross-check `stackbone docs agent-yaml`                                                         |
| Build           | Buildpack timeout           | Re-run with `--verbose` to see the build log; common cause is heavy install (`pnpm install --frozen-lockfile` should be fast) |
| Scan (Trivy)    | Blocking CVE in base image  | Buildpack pins the base; if a CVE landed, upgrade the SDK (which pulls the new buildpack)                                     |
| Sign (cosign)   | Sigstore unreachable        | Transient — retry. If persistent, Sigstore status page.                                                                       |
| Push (registry) | Auth error                  | Re-run `stackbone login` — registry credentials are short-lived and refreshed from your session                               |
| Register        | Conflict on version         | `stackbone publish --version <next>` — versions are monotonic per template                                                    |

---

## Secrets / config / connections

| Symptom                                                                                     | Diagnostic                                                                                                                                             |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `client.secrets.get('FOO')` returns `data: null, error: null`                               | The secret isn't set — check Studio's Secrets tab for that agent                                                                                       |
| `client.config.get<T>()` returns the wrong shape                                            | The org member edited the config to something that doesn't match your `defineAgent({ schema })` constraints — surface the error so they can correct it |
| `client.connections.from('notion').getToken()` returns `error: 'connection_not_authorized'` | The org never completed OAuth — direct the user to Studio's Connections tab                                                                            |
| `client.connections.from('gdrive').getToken()` returns an expired token                     | Refresh is automatic; if it fails, the OAuth refresh token was revoked — re-authorize                                                                  |

---

## Tier / billing — 402 responses

| Body's `error.code`               | Meaning                                    | Action                                                            |
| --------------------------------- | ------------------------------------------ | ----------------------------------------------------------------- |
| `tier_quota_exceeded`             | Period credit bundle spent                 | Owner upgrades the org's tier (`stackbone docs tiers` for limits) |
| `installed_agents_cap_reached`    | Org at `installed agents` cap for its tier | Owner uninstalls an agent or upgrades                             |
| `members_cap_reached`             | Org at members cap (`free` = 1)            | Owner upgrades to `starter` or higher                             |
| `published_templates_cap_reached` | Creator at `published agent_templates` cap | Owner upgrades                                                    |

These are **not retryable** — surface `error.message` and `error.nextActions` to the user verbatim.

---

## What this skill does NOT cover

- **Performance tuning** of the agent (cold start, Fly Machine sizing, scale-to-zero pauses). That's a V1+ feature with its own metrics surface.
- **Cost analysis** beyond the 402 codes above. The Studio Costs panel covers per-agent breakdown.
- **Custom OTel queries** against Loki/Tempo. The V1+ observability surface will expose them; for now Studio's Logs tab is the supported path.
