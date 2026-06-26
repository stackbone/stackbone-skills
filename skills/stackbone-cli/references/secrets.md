# `stackbone secrets`

Manage the environment-scoped secrets bound to the targeted agent installation. Targets the local-dev install by default; override with `--agent <id>`. `remove` is **destructive** and requires `--yes`.

> **Security: the CLI never reveals a plaintext secret.** There is deliberately **no** `secrets get` / reveal verb — reading a value is a human-only Studio action behind a magic-link re-auth challenge. `list` only ever prints the masked `value_preview` placeholder.

## Subcommands

| Command                             | Description                                                                                         |
| ----------------------------------- | --------------------------------------------------------------------------------------------------- |
| `stackbone secrets list`            | Secret names + masked previews + `last_rotated_at`. Never the value.                                |
| `stackbone secrets set <name>`      | Create or rotate (idempotent on name). Value from `--value <v>` or stdin. `--description` optional. |
| `stackbone secrets remove <name>` ✱ | Delete a secret. Requires `--yes`.                                                                  |

```sh
stackbone secrets list --json
stackbone secrets set OPENAI_API_KEY --value sk-... --json
echo -n "sk-..." | stackbone secrets set OPENAI_API_KEY --json   # value via stdin
stackbone secrets remove OLD_KEY --yes --json
```

Piping a value via stdin trims a single trailing newline (so `echo secret | stackbone secrets set NAME` does not smuggle the line terminator into the secret). An empty value is rejected CLI-side.

## Exit codes

| Code | When                                                        |
| ---- | ----------------------------------------------------------- |
| 0    | Success                                                     |
| 1    | Network / validation (blank name, empty value)              |
| 2    | Not authenticated                                           |
| 3    | No install resolved — pass `--agent` or run `stackbone dev` |
| 4    | Secret name not found (on `remove`)                         |
| 5    | `remove` without `--yes`                                    |
