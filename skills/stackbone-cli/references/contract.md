# `stackbone contract`

Inspect the Stackbone Agent Protocol contract the targeted installation advertises, and validate the local project against it. Targets the local-dev install by default; override with `--agent <id>`. Every verb is read-only (no `--yes`).

## Subcommands

| Command                           | Description                                                                                                                                          |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `stackbone contract show`         | Full contract: version, minSupported, build, runtimeUrl, capabilities.                                                                               |
| `stackbone contract capabilities` | The capabilities the install reports (e.g. `queues.jobs`, `storage.s3`). **Derived** from the contract handshake — no separate endpoint.             |
| `stackbone contract validate`     | Validate the local `agent.yaml` against the advertised contract (or the local emulator contract when no project is linked). Useful before a publish. |

```sh
stackbone contract show --json
stackbone contract capabilities --json
stackbone contract validate --json   # { ok, violations, deviations, contractVersion, source }
```

> Per-agent input/output schemas are **per workflow** now — inspect them with `stackbone workflows schema <name>` (see [references/workflows.md](workflows.md)), not a single agent-level `/invoke` schema.

`validate` is CLI-native (no HTTP endpoint of its own): it reads `agent.yaml`, fetches the target's contract (or builds the local emulator contract when there is no install to target), then checks that (a) the contract version is still supported and (b) every capability the manifest declares is advertised. Today's starters rarely declare `capabilities`, so a missing array is recorded in `deviations` rather than failing — only real version/capability mismatches are `violations`. `validate` exits **1** when there are violations.

> Capabilities use the `<module>.<role>` naming (e.g. `queues.jobs`), not `<module>.<implementation>`.

## Exit codes

| Code | When                                                                       |
| ---- | -------------------------------------------------------------------------- |
| 0    | Read OK, or `validate` passed                                              |
| 1    | Network error, or `validate` found violations (incompatible)               |
| 2    | Not authenticated                                                          |
| 3    | No `agent.yaml` (on `validate`), or no install resolved for the read verbs |
| 4    | Install not found                                                          |
