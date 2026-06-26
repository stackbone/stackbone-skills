# `stackbone hitl`

Drive the human-in-the-loop approvals inbox of the targeted agent installation. Targets the local-dev install by default; override with `--agent <id>`. `list` is paginated (`--limit` + `--cursor`). `approve` and `reject` decide a pending approval — both **destructive**, both require `--yes`.

> In the SDK, a workflow opens the pause with `requestApproval()` from `@stackbone/sdk/workflow` (backed by the `stackbone.approval` inbox surface). The CLI verb group is named `hitl` after the product concept — same approvals, different entrypoint. See the **stackbone** skill for the SDK side.

## Subcommands

| Command                             | Description                                                                                                   |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `stackbone hitl list`               | Approvals inbox. `--status pending\|approved\|rejected\|timed_out\|cancelled`, `--limit` (1-100), `--cursor`. |
| `stackbone hitl get <hitlId>`       | One approval with its audit trail of past decisions.                                                          |
| `stackbone hitl approve <hitlId>` ✱ | Approve a pending approval. `--reason <note>` maps to the decision comment. Requires `--yes`.                 |
| `stackbone hitl reject <hitlId>` ✱  | Reject a pending approval. `--reason <note>` optional. Requires `--yes`.                                      |

```sh
stackbone hitl list --status pending --json
stackbone hitl get apr_01HX... --json
stackbone hitl approve apr_01HX... --reason "looks good" --yes --json
stackbone hitl reject apr_01HX... --yes --json
```

Payload editing (the web-UI `editedPayload` affordance) is intentionally **not** exposed here — the CLI only approves/rejects.

## Exit codes

| Code | When                                                                                       |
| ---- | ------------------------------------------------------------------------------------------ |
| 0    | Success                                                                                    |
| 1    | Network / validation (bad `--status`, blank id), or deciding an approval no longer pending |
| 2    | Not authenticated                                                                          |
| 3    | No install resolved — pass `--agent` or run `stackbone dev`                                |
| 4    | Approval id not found                                                                      |
| 5    | `approve` / `reject` without `--yes`                                                       |
