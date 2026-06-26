# `stackbone prompts`

Manage the versioned, named-by-key prompt catalog on the targeted agent installation. Targets the local-dev install by default; override with `--agent <id>`. `remove` and `rollback` are **destructive** and require `--yes`.

> The catalog is append-only: each key holds a chain of immutable versions, the highest is the live head, and `update` appends a new version when `content` changes; `rollback` copies an older version forward to a new head. `get` / `preview` resolve the current version by default, or a pinned `--version`.

## Subcommands

| Command                                            | Description                                                                                                                                                |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `stackbone prompts list`                           | Every prompt at its current version.                                                                                                                       |
| `stackbone prompts get <key>`                      | Print a prompt. `--version <n>` pins a version (else current).                                                                                             |
| `stackbone prompts create <key> --name <label>`    | Register at version 1. Content via `--template <inline>`, `--file <path>`, or stdin. `--description`, `--metadata <json>` optional.                        |
| `stackbone prompts update <key>`                   | Append a version and/or patch the head. New content via `--template`/`--file`; or just `--name`/`--description`/`--metadata`. At least one field required. |
| `stackbone prompts remove <key>` ✱                 | Soft-delete a prompt. Requires `--yes`.                                                                                                                    |
| `stackbone prompts versions <key>`                 | Version history (newest first, up to 200).                                                                                                                 |
| `stackbone prompts rollback <key> --version <n>` ✱ | Promote a prior version to the live head. Requires `--yes`.                                                                                                |
| `stackbone prompts preview <key>`                  | Server-side compile against a `--vars <file.json>` of values (defaults `{}`). `--version` pins. Reports a missing `{{var}}` cleanly.                       |

```sh
stackbone prompts create welcome_email --name "Welcome email" --template "Hi {{name}}" --json
stackbone prompts update welcome_email --file ./welcome.txt --json
stackbone prompts preview welcome_email --vars ./vars.json --json   # { ok, version, output, missingVar }
stackbone prompts rollback welcome_email --version 2 --yes --json
```

`--template` (inline literal) and `--file` (path) are mutually exclusive. `preview --vars` takes a **path to a JSON file**, not inline JSON. A `{ ok: false, missingVar }` from `preview` is a clean 200 outcome, not an error.

## Exit codes

| Code | When                                                                                                        |
| ---- | ----------------------------------------------------------------------------------------------------------- |
| 0    | Success (including a `preview` that reports a missing var)                                                  |
| 1    | Network / validation (both `--template` and `--file`, nothing to update, bad `--version`, missing `--name`) |
| 2    | Not authenticated                                                                                           |
| 3    | No install resolved — pass `--agent` or run `stackbone dev`                                                 |
| 4    | Prompt key (or pinned version) not found                                                                    |
| 5    | `remove` / `rollback` without `--yes`                                                                       |
