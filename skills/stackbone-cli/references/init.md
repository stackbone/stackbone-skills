# `stackbone init`

Scaffold a new Stackbone workspace **and link it** to your organization. `init` is workspace-first: it emits the project shell and, for **every** `--with` kind, registers the workspace in the control plane and writes the single `.stackbone/project.json` link — so you must be signed in first (`stackbone login`). There is no offline `init`.

## Synopsis

```sh
stackbone init [dir] [--with empty|agent|workflow|workflow-agent] [--name <ws>] [--json] [--yes] [--force]
```

## Behavior

1. Pick the target directory:
   - With a `dir` positional: writes the workspace into `./<dir>/`.
   - Without `dir`: writes into a subdirectory derived from the workspace name (so a bare `init` from a parent folder doesn't pollute it).
2. Emit the workspace shell — a multi-piece project (`deep-agents/` + `workflows/`):
   - `deep-agents/` — one deep agent per subfolder (each a `deep-agents/<name>/index.ts` default-exporting `defineDeepAgent(...)`).
   - `workflows/` — durable workflows (`workflows/<name>.workflow.ts`).
   - `package.json` (with `deepagents`, `@langchain/*`, `workflow` and `@stackbone/sdk` pinned at the root — the process must resolve ONE copy of each), `tsconfig.json`, an `.npmrc` (hoisted `node_modules` for the same reason), `.gitignore`, and a `README`.
   - Best-effort: install the Stackbone agent skills (`stackbone`, `stackbone-cli`, `stackbone-debug`) into the directories your coding tools recognize.
3. Scaffold the optional first piece chosen by `--with` on top of the shell:
   - `empty` (default) — shell only.
   - `agent` — one deep agent at `deep-agents/<name>/index.ts`.
   - `workflow` — one durable workflow at `workflows/<name>.workflow.ts`.
   - `workflow-agent` — a deep agent plus a workflow already wired to call it.
   - With a TTY and no `--with`, `init` shows an interactive picker for the first piece.
4. **Link the workspace.** For every `--with` kind, `init` registers the workspace's identity (an agent named after the workspace) and its local-dev installation, then writes `.stackbone/project.json`. That single link is what `stackbone dev`, `stackbone publish` and the management commands read. Because linking hits the control plane, `init` fails fast with exit `2` if you are not signed in — before any files are written.

## Flags

| Flag                                            | Description                                                                                                                 |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `--with empty\|agent\|workflow\|workflow-agent` | The optional first piece to scaffold on top of the shell. Defaults to `empty` (shell only). Every kind links the workspace. |
| `--name <ws>`                                   | The workspace name (and the default name for the first piece). The `dir` positional sets the target subdirectory.           |
| `--force`                                       | Overwrite existing files in the target directory.                                                                           |
| `--json`                                        | Emit a JSON envelope.                                                                                                       |
| `--yes`                                         | Skip "directory exists, overwrite?" confirmations.                                                                          |

> There is no `--starter` / `--template` / `--slug` / `--description` on `init` anymore. Starters are gone from `init`; passing `--starter` or `--template` prints a migration message and exits non-zero. Per-piece templates now live on `add` (`--template`).

## Examples

```sh
# Workspace shell, linked to your org, written to ./my-thing/ (needs login)
stackbone init my-thing

# Interactive: pick the first piece, then writes + links the workspace
stackbone init my-thing --name my-thing

# Start with an agent (the agent is the workspace's linked identity)
stackbone init my-thing --with agent --name support-bot

# Start with a durable workflow (still links the workspace)
stackbone init my-thing --with workflow --json --yes
```

## After `init`

```sh
cd my-thing
npm install              # or pnpm install
stackbone dev            # boots the local stack + Studio on 127.0.0.1:4242
```

The workspace is already linked, so `stackbone dev` boots straight away — it reads the `.stackbone/project.json` that `init` wrote. To attach a **fresh clone** (where `.stackbone/` is gitignored and therefore absent), run `stackbone link`.

### JSON envelope (`--json`)

```jsonc
{
  "schema_version": "…",
  "workspace": { "name": "…", "dir": "…" },
  "with": "empty|agent|workflow|workflow-agent",
  "files_written": ["…"],
  "agent": { "id": "…", "slug": "…", "name": "…" },
  "local_dev_installation": { "id": "…", "organization_slug": "…" },
  "skills_install": "…",
}
```

The `agent` and `local_dev_installation` keys are **always** present — `init` links the workspace for every `--with` kind.

## `stackbone add`

Once a workspace exists, grow it one piece at a time with `stackbone add deep-agent|workflow|workflow-agent <name>`. `add` runs **inside a workspace** and is **100% offline**: the pieces you add are members of the already-linked workspace, not separate registrations, so they never touch the network and never re-write `.stackbone/project.json`. See [references/add.md](add.md) for the full reference.

## Discovery by convention

The workspace registry is derived from the files on disk — there is no hand-maintained registry. `stackbone dev` and `stackbone publish` both read the same convention:

- **Agents** — every `deep-agents/<name>/` folder containing an `index.ts`. The folder basename is the agent's name.
- **Workflows** — every `workflows/<name>.workflow.ts`. The workflow name is the file basename without the `.workflow.ts` suffix; the exported function is `<camelCase(name)>Workflow` (e.g. `qualify-lead.workflow.ts` exports `qualifyLeadWorkflow`).
- **`stackbone.config.ts`** (default-exporting `defineWorkspace({ agents: [], workflows })`) is an **optional override for the workflow list only** — deep agents are always discovered from their folders. Most projects need no `stackbone.config.ts` at all.

## Exit codes

| Code | When                                                                                             |
| ---- | ------------------------------------------------------------------------------------------------ |
| 0    | Scaffold completed and workspace linked                                                          |
| 1    | Name collision without `--force`, unknown `--with` value, unknown `--template`, disk write error |
| 2    | Not authenticated (`init` links the workspace, so it needs a signed-in session for every kind)   |

## Common mistakes

- **Confusing `dir` and `--name`.** The positional is the target directory (`stackbone init my-thing`); `--name <ws>` is the workspace name (and default first-piece name). They are different things.
- **Passing `--starter` / `--template` to `init`.** Both are gone from `init` — they print a migration message and exit non-zero. `--template` survives only as a flag on `stackbone add workflow <name> --template <t>`.
- **Running `init` while signed out.** `init` links the workspace to your org, so it needs a session — run `stackbone login` first (exit code `2` otherwise). There is no offline `init`.
- **Forgetting `npm install`.** The scaffolded project has a `package.json` with dependencies; the CLI does not install them.
- **Committing `.stackbone/`.** `init` gitignores it; sharing it leaks the per-developer local-dev install id and makes `stackbone dev` sessions collide. A fresh clone re-links with `stackbone link`.
