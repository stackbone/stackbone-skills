# `stackbone config`

Manage the agent's `AGENT_CONFIG` document on the targeted installation, and regenerate the local config types. Targets the local-dev install by default; override with `--agent <id>` — except `types`, which is **local codegen** and takes no `--agent`. `rollback` is **destructive** and requires `--yes`.

> Config is **one versioned JSON document**, not a bag of key/value pairs. Every `set` appends a new version; the highest version is the active config; `rollback` copies a prior version forward (history is never edited). So the surface is `get` / `set` / `versions` / `rollback` — there is no per-key list/get/set/remove.

## Subcommands

| Command                                     | Description                                                                                                                                                                    |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `stackbone config get`                      | Print the active document (value, version, author). Version `0` means none set yet.                                                                                            |
| `stackbone config set`                      | Persist a new version. Reads a JSON **object** from `--file <path>` or stdin. A no-op write reports `unchanged`.                                                               |
| `stackbone config versions`                 | List recent versions (newest first, up to 100), noting any rollback origin.                                                                                                    |
| `stackbone config rollback --version <n>` ✱ | Roll the active config back to a prior version. Requires `--yes`.                                                                                                              |
| `stackbone config types`                    | **Local.** Regenerate `.stackbone/config.d.ts` from this project's `config.schema.ts` so `stackbone.config` reads are typed. No `--agent`, no network. `--cwd <dir>` optional. |

```sh
stackbone config get --json
stackbone config set --file ./config.json --json     # { config: { version }, unchanged }
echo '{"feature":{"newFlow":true}}' | stackbone config set --json
stackbone config versions --json
stackbone config rollback --version 3 --yes --json
stackbone config types --json                        # { file, changed, removed, hasSchema }
```

The value must be a JSON object (not a primitive/array), validated CLI-side. `config types` mirrors the codegen `stackbone dev` runs at boot (and on every `config.schema.ts` edit); with no `config.schema.ts`, any stale `config.d.ts` is removed and reads stay loose.

## Exit codes

| Code | When                                                        |
| ---- | ----------------------------------------------------------- |
| 0    | Success                                                     |
| 1    | Network / validation (non-object value, bad `--version`)    |
| 2    | Not authenticated                                           |
| 3    | No install resolved — pass `--agent` or run `stackbone dev` |
| 4    | Target version not found (on `rollback`)                    |
| 5    | `rollback` without `--yes`                                  |
