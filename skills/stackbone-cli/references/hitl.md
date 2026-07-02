# `stackbone hitl`

Drive the human-in-the-loop approvals inbox of the targeted agent installation. Targets the local-dev install by default; override with `--agent <id>`. `list` is paginated (`--limit` + `--cursor`). `approve` and `reject` decide a pending approval — both **destructive**, both require `--yes`.

> The same inbox carries **both HITL levels**: workflow pauses opened with `requestApproval()` from `@stackbone/sdk/workflow`, and **tool-level agent pauses** opened by `interruptOn` on a deep agent (topic `tool-approval for <tool> on <agent>`). Deciding either resumes the parked run server-side — for an agent pause, `approve` executes the gated tool and the resumed reply lands on the same run; `reject` surfaces your `--reason` to the model. See the **stackbone** skill's HITL doc for the SDK side.

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
