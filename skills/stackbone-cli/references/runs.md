# `stackbone runs`

Inspect and control execution runs of the targeted agent installation. Targets the local-dev install by default; override with `--agent <id>` (discover ids with `stackbone agents list`). `list` is paginated (`--limit` + `--cursor` → `{ items, nextCursor, prevCursor }`). `retry` and `cancel` are **destructive** and require `--yes`.

## Subcommands

| Command                           | Description                                                                                |
| --------------------------------- | ------------------------------------------------------------------------------------------ |
| `stackbone runs list`             | Recent runs. `--status running\|done\|failed\|interrupted`, `--limit` (1-100), `--cursor`. |
| `stackbone runs get <runId>`      | One run header: status, trigger, timing.                                                   |
| `stackbone runs retry <runId>` ✱  | Start a fresh run from the failed run's input. Requires `--yes`.                           |
| `stackbone runs cancel <runId>` ✱ | Mark a running run interrupted. Requires `--yes`.                                          |

```sh
stackbone runs list --status failed --limit 20 --json
stackbone runs get run_01HX... --json
stackbone runs retry run_01HX... --yes --json   # { run: { id, status } } — a NEW run id
stackbone runs cancel run_01HX... --yes --json
```

> There is **no** `stackbone runs steps` verb. To trace one run's detail, read its header with `stackbone runs get <runId>` and its per-run log lines with `stackbone logs tail --run <runId>`.

## Exit codes

| Code | When                                                           |
| ---- | -------------------------------------------------------------- |
| 0    | Success                                                        |
| 1    | Network / validation (bad `--status`, `--limit`, blank run id) |
| 2    | Not authenticated                                              |
| 3    | No install resolved — pass `--agent` or run `stackbone dev`    |
| 4    | Run id not found on the target                                 |
| 5    | `retry` / `cancel` without `--yes`                             |
