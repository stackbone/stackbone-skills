# `stackbone add`

Add one more piece to a workspace you already scaffolded with `stackbone init`. Three kinds:

- `stackbone add agent <name>` — one eve agent under `agents/<name>/`. Registers the agent in the control plane (needs login).
- `stackbone add workflow <name>` — one durable workflow at `workflows/<name>.workflow.ts`. Dev-only, never touches the control plane (no login).
- `stackbone add workflow-agent <name>` — the composed template: an agent **plus** a workflow already wired to call it. Registers the agent like `add agent`.

`add` only ever **writes new files**. It never edits your existing TypeScript and never edits `stackbone.config.ts`. The workspace registry is discovered from the files on disk (see [Discovery by convention](#discovery-by-convention)), so `add` has nothing to register against — there is no list to keep in sync.

## Synopsis

```sh
stackbone add agent <name>          [--template <t>] [--json] [--yes] [--force]
stackbone add workflow <name>       [--template <t>] [--calls <agent>] [--json] [--yes] [--force]
stackbone add workflow-agent <name> [--template <t>] [--json] [--yes] [--force]
```

`add` must run **inside a workspace** — a directory that already has an `agents/` folder (i.e. one `init` produced). Run it from the workspace root; the `<name>` is the piece's name, not a path.

## Flags

| Flag              | Applies to | Description                                                                                          |
| ----------------- | ---------- | ---------------------------------------------------------------------------------------------------- |
| `--template <t>`  | all        | Choose a per-piece template. An unknown template is rejected.                                        |
| `--calls <agent>` | `workflow` | Wire a step inside the new workflow that delegates a turn to that agent (the workflow→agent hybrid). |
| `--force`         | all        | Overwrite on a name collision instead of failing.                                                    |
| `--json`          | all        | Emit a JSON envelope.                                                                                |
| `--yes`           | all        | Skip confirmation prompts.                                                                           |

## Offline vs network

Only the **agent-creating** kinds touch the control plane:

| Kind             | Network? | Login? | What happens                                                                                        |
| ---------------- | -------- | ------ | --------------------------------------------------------------------------------------------------- |
| `agent`          | yes      | yes    | Registers the agent **eagerly** in the control plane and creates its own local-dev install.         |
| `workflow-agent` | yes      | yes    | Registers the agent the same way; the workflow half stays local.                                    |
| `workflow`       | no       | no     | Writes only the `.workflow.ts` file. Workflows are dev-only today, so there is nothing to register. |

`add agent` is **idempotent**: re-running `stackbone add agent <name>` resolves the same control-plane row rather than creating a duplicate.

## `stackbone add agent`

Adds `agents/<name>/` (the eve agent folder — `agent.yaml`, `agent/agent.ts`, etc.). The manifest `name:` is set to `<name>` and **must equal the folder basename** — that is how discovery finds it. Registers the agent in your active org and mints a local-dev installation so `stackbone dev` can boot it.

```sh
stackbone add agent lead-qualifier --json
# {
#   "schema_version": …,
#   "kind": "agent",
#   "name": "lead-qualifier",
#   "target_dir": "/path/to/your-workspace",  // the workspace root, not the piece folder
#   "files_written": [ … ],  // paths RELATIVE to the workspace root, e.g. "agents/lead-qualifier/agent.yaml"
#   "registered_in_config": false,
#   "control_plane_agent": { "id": …, "slug": …, "name": … },
#   "local_dev_installation": { "id": …, "organization_slug": … }
# }
```

## `stackbone add workflow`

Adds `workflows/<name>.workflow.ts`. The exported function is `<camelCase(name)>Workflow` (e.g. `qualify-lead.workflow.ts` exports `qualifyLeadWorkflow`). Never touches the control plane.

`--calls <agent>` scaffolds a step that delegates a turn to that agent, so the workflow drives an existing agent.

```sh
stackbone add workflow qualify-lead --json
# {
#   "schema_version": …,
#   "kind": "workflow",
#   "name": "qualify-lead",
#   "target_dir": "/path/to/your-workspace",  // the workspace root
#   "files_written": [ "workflows/qualify-lead.workflow.ts" ],
#   "registered_in_config": false,
#   "control_plane_agent": null
# }

# Wire the workflow to call an existing agent:
stackbone add workflow qualify-lead --calls lead-qualifier
```

## `stackbone add workflow-agent`

The composed template — it scaffolds an agent **and** a workflow already wired to call it (the `qualify-lead → lead-qualifier` pattern). Registers the agent exactly like `add agent`. The JSON envelope mirrors `add agent` but with `"kind": "workflow-agent"`.

```sh
stackbone add workflow-agent lead-qualifier --json
# { "schema_version": …, "kind": "workflow-agent", "name": …, "target_dir": …,
#   "files_written": [ … ], "registered_in_config": false,
#   "control_plane_agent": { … }, "local_dev_installation": { … } }
```

> `registered_in_config` is always `false`: `add` never writes the name into `stackbone.config.ts` (or anywhere else). The workspace is discovered from the files themselves.

## Discovery by convention

There is **no hand-maintained registry**. The workspace registry is derived from the files on disk, and `stackbone dev` + `stackbone publish` both read this same convention:

- **Agents** — every `agents/<name>/agent.yaml`. The manifest `name:` field is authoritative and **must equal the folder basename**.
- **Workflows** — every `workflows/<name>.workflow.ts`. The workflow name is the file basename without the `.workflow.ts` suffix; the exported function is `<camelCase(name)>Workflow`.
- **`stackbone.config.ts`** (default-exporting `defineWorkspace({ agents, workflows })`) is an **optional override**: if it exists it wins over the convention scan; if it is absent the workspace is discovered straight from the files. Most projects need none.

So after `add`, the new piece is picked up automatically the next time you run `stackbone dev` or `stackbone publish` — nothing edits a config, and you do not register it anywhere.

> `add` is a workspace command — it expects the `agents/` folder that `stackbone init` produces.

## Exit codes

| Code | When                                                                                              |
| ---- | ------------------------------------------------------------------------------------------------- |
| 0    | Piece written (and registered, for agent / workflow-agent)                                        |
| 1    | Name collision without `--force`, an unknown `--template`, or a generic write/control-plane error |
| 2    | Not authenticated — an agent-creating add (`agent` / `workflow-agent`) run while signed out       |
| 3    | Not in a workspace — `add` run outside a scaffolded workspace                                     |

## Common mistakes

- **Running `add` outside a workspace.** `add` needs the `agents/` folder that `stackbone init` produces (exit code 3). To start fresh, run `stackbone init` first.
- **Expecting `add` to edit your code.** `add` only writes new files; it never touches your existing TypeScript or `stackbone.config.ts`. Wire new pieces together yourself (or use `--calls` / `workflow-agent` to get a pre-wired pair).
- **A name collision.** `add agent support` when `agents/support/` already exists fails with a clear error — re-run with `--force` to overwrite.
- **Signing in for `add workflow`.** Workflows are dev-only and never hit the control plane; only `add agent` / `add workflow-agent` need a login.
- **Renaming a folder without the manifest.** Discovery keys agents on `agents/<name>/agent.yaml` where the manifest `name:` equals the folder basename — rename both together, or the agent disappears from `dev` / `publish`.

## After `add`

```sh
stackbone dev            # picks up the new agent/workflow by convention, no config edit
```

The new piece is discovered from disk, so the running workspace serves it immediately. Durable workflows, runs and HITL are exercised by `stackbone dev` (the local emulator); on a cloud installation those surfaces may still answer `404` until the cloud port lands, so develop workflows against `stackbone dev`.
