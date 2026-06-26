# `stackbone init`

Scaffold a new Stackbone workspace. Workspace-first and offline by default (`--with empty`) — it emits the project shell and runs no network call unless you ask for an agent-creating first piece.

## Synopsis

```sh
stackbone init [dir] [--with empty|agent|workflow|workflow-agent] [--name <ws>] [--json] [--yes] [--force]
```

## Behavior

1. Pick the target directory:
   - With a `dir` positional: writes the workspace into `./<dir>/`.
   - Without `dir`: writes into the current directory.
2. Emit the workspace shell — a multi-piece project (`agents/` + `workflows/`):
   - `agents/` — one eve agent per subfolder (`agents/<name>/agent.yaml`).
   - `workflows/` — durable workflows (`workflows/<name>.workflow.ts`).
   - `package.json`, `tsconfig.json`, an `.npmrc` (eve needs a hoisted `node_modules`), `.gitignore`, and a `README`.
   - Best-effort: install the Stackbone agent skills (`stackbone`, `stackbone-cli`, `stackbone-debug`) into the directories your coding tools recognize.
3. Scaffold the optional first piece chosen by `--with` on top of the shell:
   - `empty` (default) — shell only, fully offline.
   - `agent` — one eve agent under `agents/<name>/`.
   - `workflow` — one durable workflow at `workflows/<name>.workflow.ts`.
   - `workflow-agent` — an agent plus a workflow already wired to call it.
   - With a TTY and no `--with`, `init` shows an interactive picker for the first piece.
4. Only the agent-creating kinds touch the network. `--with agent` and `--with workflow-agent` register the agent eagerly in the control plane and create a local-dev installation row, so you must be signed in. `--with empty` and `--with workflow` are fully offline.

## Flags

| Flag                                            | Description                                                                                                       |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `--with empty\|agent\|workflow\|workflow-agent` | The optional first piece to scaffold on top of the shell. Defaults to `empty` (shell only, offline).              |
| `--name <ws>`                                   | The workspace name (and the default name for the first piece). The `dir` positional sets the target subdirectory. |
| `--force`                                       | Overwrite existing files in the target directory.                                                                 |
| `--json`                                        | Emit a JSON envelope.                                                                                             |
| `--yes`                                         | Skip "directory exists, overwrite?" confirmations.                                                                |

> There is no `--starter` / `--template` / `--slug` / `--description` on `init` anymore. Starters are gone from `init`; passing `--starter` or `--template` prints a migration message and exits non-zero. Per-piece templates now live on `add` (`--template`).

## Examples

```sh
# Offline workspace shell, writes to ./my-thing/ (no network call)
stackbone init my-thing

# Interactive: pick the first piece, then writes the workspace
stackbone init my-thing --name my-thing

# Start with an agent (registers it in the control plane — must be signed in)
stackbone init my-thing --with agent --name support-bot

# Start with a durable workflow (offline)
stackbone init my-thing --with workflow --json --yes
```

## After `init`

```sh
cd my-thing
npm install              # or pnpm install
stackbone dev            # boots the local stack + Studio on 127.0.0.1:4242
```

By default (`--with empty`) `init` creates no agent and makes no network call — you start from an offline workspace shell. An agent identity (and a `.stackbone/` local-dev install) only appears when you scaffold an agent-creating piece: either `init --with agent|workflow-agent`, or later via `stackbone add agent` / `stackbone add workflow-agent`. To re-attach a fresh clone, run `stackbone link`.

### JSON envelope (`--json`)

```jsonc
{
  "schema_version": "…",
  "workspace": { "name": "…", "dir": "…" },
  "with": "empty|agent|workflow|workflow-agent",
  "files_written": ["…"],
  // present only for --with agent | workflow-agent:
  "agent": { "id": "…", "slug": "…", "name": "…" },
  "local_dev_installation": { "id": "…", "organization_slug": "…" },
  "skills_install": "…",
}
```

The `agent` and `local_dev_installation` keys appear only for `--with agent | workflow-agent`.

## `stackbone add`

Add a single piece to the current workspace. `add` must run inside a workspace. It **only writes new files** — it never edits your existing TypeScript and never edits `stackbone.config.ts`. A name collision fails with a clear error; re-run with `--force` to overwrite.

```sh
stackbone add agent <name>          [--template <t>] [--json] [--yes] [--force]
stackbone add workflow <name>       [--template <t>] [--calls <agent>] [--json] [--yes] [--force]
stackbone add workflow-agent <name> [--template <t>] [--json] [--yes] [--force]
```

- **`add agent <name>`** — writes one eve agent folder under `agents/<name>/` and registers it eagerly in the control plane plus its own local-dev install, so you must be signed in. Idempotent: re-running `add agent <name>` resolves the same control-plane row.
- **`add workflow <name>`** — writes one durable workflow at `workflows/<name>.workflow.ts`. Never touches the control plane (workflows are dev-only today), so no login is required. `--calls <agent>` wires a step inside the workflow that delegates a turn to that agent (the workflow → agent hybrid).
- **`add workflow-agent <name>`** — the composed template: scaffolds an agent **and** a workflow already wired to call it (the qualify-lead → lead-qualifier pattern). Registers the agent like `add agent`.

JSON envelope (`--json`):

```jsonc
{
  "schema_version": "…",
  "kind": "agent|workflow|workflow-agent",
  "name": "…",
  "target_dir": "…",
  "files_written": ["…"],
  "registered_in_config": false,
  // for agent / workflow-agent:
  "control_plane_agent": { "id": "…", "slug": "…", "name": "…" },
  "local_dev_installation": { "id": "…", "organization_slug": "…" },
  // for workflow: "control_plane_agent": null (no local_dev_installation)
}
```

## Discovery by convention

The workspace registry is derived from the files on disk — there is no hand-maintained registry. `stackbone dev` and `stackbone publish` both read the same convention:

- **Agents** — every `agents/<name>/agent.yaml`. The manifest `name:` field is authoritative and must equal the folder basename.
- **Workflows** — every `workflows/<name>.workflow.ts`. The workflow name is the file basename without the `.workflow.ts` suffix; the exported function is `<camelCase(name)>Workflow` (e.g. `qualify-lead.workflow.ts` exports `qualifyLeadWorkflow`).
- **`stackbone.config.ts`** (default-exporting `defineWorkspace({ agents, workflows })`) is an **optional override**: if present it wins over the convention scan; if absent the workspace is discovered from the files. Most projects need no `stackbone.config.ts` at all.

## Exit codes

| Code | When                                                                                             |
| ---- | ------------------------------------------------------------------------------------------------ |
| 0    | Scaffold completed                                                                               |
| 1    | Name collision without `--force`, unknown `--with` value, unknown `--template`, disk write error |
| 2    | Not authenticated (an agent-creating `init`/`add` run while not signed in)                       |
| 3    | No project (an `add` run outside a workspace, or `dev` with no project)                          |

## Common mistakes

- **Confusing `dir` and `--name`.** The positional is the target directory (`stackbone init my-thing`); `--name <ws>` is the workspace name (and default first-piece name). They are different things.
- **Passing `--starter` / `--template` to `init`.** Both are gone from `init` — they print a migration message and exit non-zero. `--template` is now a per-piece flag on `add` (e.g. `stackbone add agent <name> --template <t>`).
- **Expecting `init` to create an agent.** The default `--with empty` is offline and creates no agent. Use `--with agent|workflow-agent` (or `stackbone add agent`) to register one.
- **Running `add` outside a workspace.** `add` must run inside a workspace (exit code `3` otherwise).
- **Forgetting `npm install`.** The scaffolded project has a `package.json` with dependencies; the CLI does not install them.
- **Committing `.stackbone/`.** `init`/`add` gitignore it; sharing it leaks the per-developer local-dev install id and makes `stackbone dev` sessions collide.
