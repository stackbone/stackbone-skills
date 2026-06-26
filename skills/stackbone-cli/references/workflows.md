# `stackbone workflows`

Inspect the durable [Workflow SDK](https://workflow-sdk.dev/docs) workflows the targeted agent installation exposes and the input/output JSON Schema each one declares, and **start one by name**. Targets the local-dev install by default; override with `--agent <id>` (discover ids with `stackbone agents list`).

> The schemas are recovered from the sibling `inputSchema` / `outputSchema` Zod exports a workflow file ships beside its `'use workflow'` function. A workflow that declares neither shows `hasSchema = false`, and `schema` returns `{ input: null, output: null }`.

## Subcommands

| Command                                                   | Description                                                                                                                                                                       |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `stackbone workflows list`                                | List the workflows the install exposes. Each item is `{ name, trigger, hasSchema }`. `◆` = declares a schema, `◇` = none. (read-only)                                             |
| `stackbone workflows schema <name>`                       | Print one workflow's `{ input, output }` JSON Schema pair. Each half is `null` when undeclared. 404 for an unknown name. (read-only)                                              |
| `stackbone workflows start <name> [--input/--input-file]` | Start a run of `<name>` by name (resolves server-side — no bundle needed). Input from `--input <json>` or `--input-file <path>`, default `{}`. Returns `{ workflowName, runId }`. |

```sh
stackbone workflows list --json                                        # { items: [{ name, trigger, hasSchema }] }
stackbone workflows schema onboarding --json                           # { schema: { input, output } }
stackbone workflows start onboarding --input '{"userId":"u1"}' --json  # { workflowName, runId }
```

## Starting a workflow

`stackbone workflows start <name>` triggers a run by name — the name resolves server-side, so the workflow need not be bundled in the CLI. Input comes from `--input <json>` (inline) or `--input-file <path>`, defaulting to `{}`; it is validated against the workflow's `inputSchema` at the frontier, so a bad shape is rejected (400 `workflow_input_invalid`) **before** any run starts.

It maps to `POST /api/workflows/:name/start`; a chat-style workflow also exposes `POST /api/workflows/:name/chat` (SSE), which you can hit directly against the `stackbone dev` server. From inside agent/workflow code, the SDK equivalents are `stackbone.workflows.start` / `stackbone.workflows.startAndWait` on the ambient client (see the **stackbone** skill). To watch the run afterwards: `stackbone runs get <runId>` + `stackbone logs tail --run <runId>`.

## Exit codes

| Code | When                                                                    |
| ---- | ----------------------------------------------------------------------- |
| 0    | Success                                                                 |
| 1    | Network / validation (blank name, bad input JSON, schema-invalid input) |
| 2    | Not authenticated                                                       |
| 3    | No install resolved — pass `--agent` or run `stackbone dev`             |
| 4    | Workflow name not found (on `schema` / `start`)                         |
