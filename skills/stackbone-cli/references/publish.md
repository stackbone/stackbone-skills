# `stackbone publish`

Build, scan, sign and push the agent template to the Stackbone registry, then register the new version of the `agent_template` row.

## Synopsis

```sh
stackbone publish [--version <semver>] [--dry-run] [--json] [--yes] [--verbose]
```

## Pipeline (what happens, in order)

1. **Validate** `agent.yaml` against the schema. Failed validation aborts before any network I/O.
2. **Validate** that the project is linked (`.stackbone/project.json` exists and points to a real `agent_template`).
3. **Resolve version**. With `--version`, uses it directly. Without, prompts (interactive) or defaults to `patch` bump (`--yes`).
4. **Buildpack**. The platform-managed buildpack runs `pnpm install --frozen-lockfile` and produces an OCI image using the wrapper as the entrypoint. There is no per-project `Dockerfile` unless `runtime.dockerfile` is declared.
5. **Trivy scan**. CVEs at severity `HIGH` or `CRITICAL` in the resolved tree block the publish (`exit 1`). The base image is platform-managed and pinned — most CVEs come from the project's own `dependencies`.
6. **cosign sign**. The image is signed with sigstore against the Stackbone signing chain.
7. **Push** to `registry.fly.io/stackbone-<org-slug>/<agent-template>:<version>`.
8. **Register**. A new row in `stackbone_platform.agent_template_version` is created with `{ version, image_ref, manifest_yaml, created_at }`. The template's `latest_version` pointer is updated.
9. **Notify**. Existing installs (`agent`s with auto-update enabled) start receiving the new version on their next invocation; manual-update installs see a notice in Studio.

## Flags

| Flag                 | Description                                                                                                                             |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `--version <semver>` | Explicit version (e.g. `0.3.0`). Versions are strictly monotonic per template; replays / downgrades are rejected.                       |
| `--dry-run`          | Run steps 1–6 (validate, buildpack, scan, sign) without pushing or registering. Use to confirm a publish will succeed before opting in. |
| `--json`             | Emit envelope with the build log paths and the new version metadata.                                                                    |
| `--yes`              | Skip "bump version?" confirmation; defaults to `patch` bump.                                                                            |
| `--verbose`          | Stream the buildpack log. Critical for diagnosing build failures.                                                                       |

## Examples

```sh
# Interactive — picks version, asks before pushing
stackbone publish

# Non-interactive, explicit version
stackbone publish --version 0.3.0 --yes --json

# Verify the publish will succeed before committing
stackbone publish --dry-run --verbose

# CI / agent shell
stackbone publish --version "${VERSION}" --yes --json --verbose
```

## What gets registered

```json
{
  "template_id": "tpl_01HX...",
  "version": "0.3.0",
  "image_ref": "registry.fly.io/stackbone-acme/contract-reviewer:0.3.0",
  "manifest_yaml": "<the agent.yaml at publish time>",
  "capabilities": ["rag", "hitl", "queues"],
  "pricing": { "model": "one_time_fee", "amount_eur": 49 },
  "created_at": "2026-05-22T10:42:18Z"
}
```

The `agent.yaml` at publish time is frozen into the row — later edits don't retroactively affect existing installs.

## What does NOT happen at publish

- **No DB migrations run** on existing installs. Migrations are bundled in the image and applied on next invocation by the wrapper's startup hook (or manually via `stackbone db migrate up --agent <id>`).
- **No secrets/config migration**. If you renamed a `client.secrets.get('OLD')` to `client.secrets.get('NEW')`, the org members need to set `NEW` themselves; an old missing secret returns `error: 'secret_not_found'`.
- **No automatic rollback**. If the new version crashes immediately on existing installs, the platform falls back to the previous version after N failed invocations — but the new version stays the `latest_version` pointer until you publish another.
- **No backend changes**. `agent.yaml.events.subscribes` adds/removes are applied; new subscriptions start receiving events from publish time forward, no replay of historical events.

## Pre-publish checklist

Before running `publish` on a production-bound template:

- [ ] `stackbone dev` runs without errors against the local emulator.
- [ ] `defineAgent({ schema })` covers every shape the agent accepts.
- [ ] Migrations under `migrations/` are committed and tested.
- [ ] `agent.yaml.capabilities` includes every module the code uses (`hitl`, `rag`, `queues`, `connections`).
- [ ] `agent.yaml.pricing` is set to the intended value (publishing with `one_time_fee.amount_eur: 0` lists the template as free).
- [ ] No `console.log()` in hot paths; use `client.observability.log()` instead.

## Exit codes

| Code | When                                                              |
| ---- | ----------------------------------------------------------------- |
| 0    | Published                                                         |
| 1    | Validation failed / buildpack error / Trivy block / network error |
| 2    | Not authenticated                                                 |
| 3    | Not linked                                                        |
| 5    | Tier quota: `published_templates_cap_reached`                     |

## Common failure modes

| Symptom                                                | Cause                                     | Resolution                                                                   |
| ------------------------------------------------------ | ----------------------------------------- | ---------------------------------------------------------------------------- |
| `validation_failed: agent.yaml`                        | Schema mismatch                           | `stackbone docs agent-yaml` and cross-check                                  |
| `buildpack_timeout`                                    | `pnpm install` slow / postinstall scripts | `--verbose` + look at the timing of the slow step; consider trimming devDeps |
| `trivy_blocked: critical CVE in <pkg>`                 | A dep ships a known critical CVE          | Upgrade the dep; if upstream hasn't patched, use `package.json.overrides`    |
| `version_conflict: 0.3.0 already published`            | Replaying a version                       | `--version 0.3.1` (versions are strictly monotonic)                          |
| `tier_quota_exceeded: published_templates_cap_reached` | Org at template cap for its tier          | Owner upgrades the tier                                                      |
| `cosign_signing_failed: sigstore unreachable`          | Transient Sigstore outage                 | Retry. If persistent, check status.sigstore.dev                              |

## Common mistakes

- **Publishing from a dirty working tree.** Uncommitted changes get baked into the image. Commit (or stash) first.
- **Bumping `package.json.version` instead of using `--version`.** The CLI ignores `package.json.version` — the source of truth is the `--version` flag or the interactive prompt.
- **Forgetting to apply migrations after publish.** The wrapper's startup hook is best-effort; for sensitive migrations, apply manually after publish: `stackbone db migrate up --all --agent <id>`.
