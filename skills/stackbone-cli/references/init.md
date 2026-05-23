# `stackbone init`

Scaffold a new Stackbone agent project from a starter.

## Synopsis

```sh
stackbone init [name] --starter <slug> [--here] [--json] [--yes]
```

## Behavior

1. Resolve the starter `<slug>` against the bundled starters list (the CLI ships them; see `stackbone list --starters`).
2. Pick the target directory:
   - With a `name` positional: writes into `./<name>/`.
   - Without `name`: nests the scaffold into a slugified subdirectory of the current dir (e.g. `stackbone init --starter ai` from `~/Projects/` creates `~/Projects/starter-ai/`). This is intentional — it avoids accidentally polluting the current directory.
   - With `--here`: writes into the current directory if it's empty. Errors if the directory has tracked files.
3. Copy the starter files recursively.
4. Find-and-replace every occurrence of the starter's `package.json.name` (e.g. `starter-ai`) with the new project name in the entire copied tree (README, `package.json`, `agent.yaml`, source). This is global on purpose — the starter is runnable with its real name as a reference repo, and the rename produces a runnable target project.
5. Append `.stackbone/`, agent-skill directories (`.claude/`, `.cursor/`, ...) and `node_modules/` to `.gitignore`.
6. Install the Stackbone agent skills (`stackbone`, `stackbone-cli`, `stackbone-debug`) into the per-agent directories the project's coding tools recognize.

## Flags

| Flag               | Description                                                                                                                             |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| `--starter <slug>` | **Required.** Starter slug. Discoverable via `stackbone list --starters`.                                                               |
| `--here`           | Write into the current directory (must be empty). Mutually exclusive with the `name` positional.                                        |
| `--json`           | Emit JSON envelope. Required when `name` is omitted and you also want non-interactive — the slug-from-current-dir behavior runs anyway. |
| `--yes`            | Skip "directory exists, overwrite?" confirmations.                                                                                      |

## Examples

```sh
# Interactive, picks org + writes to ./my-thing/
stackbone init my-thing --starter rag

# Non-interactive, no name → nests as ./starter-ai-<timestamp>/ or similar
stackbone init --starter ai --json --yes

# Write into the current empty dir
mkdir my-thing && cd my-thing && stackbone init --here --starter db

# Discover available starters first
stackbone list --starters --json | jq '.data[] | {slug, label, hint}'
```

## What's in the resulting directory

Top-level layout of a freshly-scaffolded project (using the `hello-world` starter as the simplest example):

```
my-thing/
├── .claude/skills/stackbone/          ← auto-installed skill
├── .claude/skills/stackbone-cli/      ← auto-installed skill
├── .claude/skills/stackbone-debug/    ← auto-installed skill
├── .gitignore                         ← includes .stackbone/, .claude/skills/, node_modules/
├── README.md                          ← starter README, find-replaced with the new name
├── agent.yaml                         ← starter manifest with name = "my-thing"
├── package.json                       ← name = "my-thing"
├── project.json                       ← Nx project descriptor (if monorepo)
├── tsconfig.json
├── vitest.config.mts
├── src/
│   └── index.ts                       ← defineAgent({ invoke })
└── test/
```

## After `init`

```sh
cd my-thing
pnpm install              # or npm install
stackbone dev             # boots the local emulator + Studio at :4242
```

If you want to publish later, the project needs to be linked first:

```sh
stackbone link            # interactive: pick org + assign a template name
```

`link` writes `.stackbone/project.json` with `{ organizationId, agentTemplateId, localDevInstallationId }`. After that, `stackbone publish` is non-interactive.

## Exit codes

| Code | When                                                                                             |
| ---- | ------------------------------------------------------------------------------------------------ |
| 0    | Scaffold completed                                                                               |
| 1    | Disk write error, find-and-replace conflict, network error fetching starter                      |
| 2    | Not authenticated (only required if the starter needs a private dependency or template metadata) |
| 4    | `--starter <slug>` not found                                                                     |

## Common mistakes

- **Passing `--name` instead of a positional.** The positional is unnamed: `stackbone init my-thing --starter ai`, not `stackbone init --name my-thing --starter ai`.
- **Forgetting `pnpm install`.** The scaffolded project has a `package.json` with dependencies; the CLI does not install them.
- **Renaming the directory after `init`.** The find-and-replace already burned the new name into `package.json` and `agent.yaml`. Renaming the directory works for git but leaves a stale `package.json.name` — fix by editing both files.
- **Committing `.stackbone/`.** The `init` adds it to `.gitignore`; if someone disables that, the per-developer local-dev install ID gets shared and `stackbone dev` sessions collide.
