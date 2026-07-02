# `stackbone add`

Add one more piece to a workspace you already scaffolded (and linked) with `stackbone init`. Three kinds:

- `stackbone add deep-agent <name>` (alias: `add agent <name>`) — one deep agent at `deep-agents/<name>/index.ts`.
- `stackbone add workflow <name>` — one durable workflow at `workflows/<name>.workflow.ts`. `--calls <agent>` wires a step that delegates a turn to an agent.
- `stackbone add workflow-agent <name>` — the composed template: a deep agent **plus** a workflow already wired to call it.

`add` is **100% offline**. The workspace was already linked to the control plane by `stackbone init`; the pieces you add are **members of that one workspace**, not separate registrations, so `add` makes no network call, needs no login, and never writes `.stackbone/project.json`. It only ever **writes new files** — it never edits your existing TypeScript and never edits `stackbone.config.ts` (one exception: the agent kinds **merge missing dependency pins** into the root `package.json`, never overwriting a version you pinned). The workspace registry is discovered from the files on disk (see [Discovery by convention](#discovery-by-convention)), so there is nothing to register against and no list to keep in sync.

## Synopsis

```sh
stackbone add deep-agent <name>     [--json] [--yes] [--force]      # alias: add agent <name>
stackbone add workflow <name>       [--template <t>] [--calls <agent>] [--json] [--yes] [--force]
stackbone add workflow-agent <name> [--json] [--yes] [--force]
```

`add` must run **inside a workspace** — a directory `stackbone init` produced (it recognizes the `deep-agents/` / `workflows/` folders or a `package.json`). Run it from the workspace root; the `<name>` is the piece's name, not a path.

## Flags

| Flag              | Applies to | Description                                                                                                              |
| ----------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------ |
| `--template <t>`  | `workflow` | Choose a workflow template. An unknown template is rejected.                                                             |
| `--calls <agent>` | `workflow` | Wire a step inside the new workflow that delegates a turn to that agent via `callDeepAgent` (the workflow→agent hybrid). |
| `--force`         | all        | Overwrite on a name collision instead of failing.                                                                        |
| `--json`          | all        | Emit a JSON envelope.                                                                                                    |
| `--yes`           | all        | Skip confirmation prompts.                                                                                               |

## `stackbone add deep-agent`

Writes **one file** — `deep-agents/<name>/index.ts` (inline system prompt, one example LangChain tool, default export `defineDeepAgent(...)`) — and merges the runtime dependency pins (`deepagents`, `@langchain/core`, `@langchain/langgraph`, `@langchain/langgraph-checkpoint-postgres`, `@langchain/openai`, `zod`) into the **root** `package.json`. Deep agents carry no per-agent `package.json`: their runtime deps must live at the root so the process resolves ONE copy. Offline: no registration, no login. Run your package manager's install after the first `add` so the new pins land in `node_modules`.

```sh
stackbone add deep-agent lead-qualifier --json
# {
#   "schema_version": …,
#   "kind": "deep-agent",
#   "name": "lead-qualifier",
#   "target_dir": "/path/to/your-workspace",  // the workspace root, not the piece folder
#   "files_written": [ "deep-agents/lead-qualifier/index.ts" ],
#   "registered_in_config": false,
#   "control_plane_agent": null   // add is offline — the workspace is the linked unit
# }
```

## `stackbone add workflow`

Adds `workflows/<name>.workflow.ts`. The exported function is `<camelCase(name)>Workflow` (e.g. `qualify-lead.workflow.ts` exports `qualifyLeadWorkflow`). `--calls <agent>` scaffolds a step that delegates a turn to that agent with `callDeepAgent` from `@stackbone/sdk/workflow`, so the workflow drives an existing agent.

```sh
stackbone add workflow qualify-lead --json
# { "schema_version": …, "kind": "workflow", "name": "qualify-lead",
#   "target_dir": "/path/to/your-workspace",
#   "files_written": [ "workflows/qualify-lead.workflow.ts" ],
#   "registered_in_config": false, "control_plane_agent": null }

# Wire the workflow to call an existing agent:
stackbone add workflow qualify-lead --calls lead-qualifier
```

## `stackbone add workflow-agent`

The composed template — it scaffolds a deep agent **and** a workflow already wired to call it (the `qualify-lead → lead-qualifier` pattern). Offline, like `add deep-agent`. The JSON envelope mirrors the others with `"kind": "workflow-agent"`.

> `registered_in_config` is always `false` and `control_plane_agent` is always `null`: `add` never writes the name into `stackbone.config.ts` (or anywhere else) and never registers a control-plane agent. The workspace is discovered from the files themselves, and its single control-plane identity was set by `init`.

## Discovery by convention

There is **no hand-maintained registry**. The workspace registry is derived from the files on disk, and `stackbone dev` + `stackbone publish` both read this same convention:

- **Agents** — every `deep-agents/<name>/` folder containing an `index.ts`. The folder basename is the agent's name.
- **Workflows** — every `workflows/<name>.workflow.ts`. The workflow name is the file basename without the `.workflow.ts` suffix; the exported function is `<camelCase(name)>Workflow`.
- **`stackbone.config.ts`** (default-exporting `defineWorkspace({ agents: [], workflows })`) is an **optional override for the workflow list only** — deep agents are always discovered from their folders.

So after `add`, the new piece is picked up automatically the next time you run `stackbone dev` or `stackbone publish` — nothing edits a config, and you do not register it anywhere.

## Exit codes

| Code | When                                                                             |
| ---- | -------------------------------------------------------------------------------- |
| 0    | Piece written                                                                    |
| 1    | Name collision without `--force`, an unknown `--template`, or a disk write error |
| 3    | Not in a workspace — `add` run outside a scaffolded workspace                    |

`add` never returns exit `2` (auth): it is offline and needs no session.

## Common mistakes

- **Running `add` outside a workspace.** `add` needs the folders that `stackbone init` produces (exit code 3). To start fresh, run `stackbone init` first.
- **Expecting `add` to link or register.** `add` is offline — `init` already linked the workspace, and added pieces are members of it. If a fresh clone has no `.stackbone/project.json` (it is gitignored), re-link with `stackbone link`, not `add`.
- **Skipping the install after the first agent.** `add deep-agent` pins `deepagents` + `@langchain/*` in the root `package.json`; run `npm install` / `pnpm install` so they resolve before `stackbone dev`.
- **A name collision.** `add deep-agent support` when `deep-agents/support/` already exists fails with a clear error — re-run with `--force` to overwrite.
- **Adding a per-agent `package.json`.** Deep agents must not carry one — a nested `node_modules` can ship a second SDK copy and split the invocation context.

## After `add`

```sh
stackbone dev            # picks up the new agent/workflow by convention, no config edit
```

The new piece is discovered from disk, so the running workspace serves it immediately (exception: the **first** deep agent in a folder-less workspace needs one dev restart — the watcher arms at boot). Durable workflows, runs and HITL are exercised by `stackbone dev` (the local emulator); on a cloud installation those surfaces may still answer `404` until the cloud port lands, so develop workflows against `stackbone dev`.
