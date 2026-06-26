# Agent guidance — Stackbone Skills repo

This document helps coding agents work effectively when **modifying** the skills in this repository (not when consuming them — for that, the skills speak for themselves).

## Repository structure

```
stackbone-skills/
├── .claude-plugin/plugin.json   # marketplace manifest for Claude Code
├── README.md                    # human-facing overview
├── AGENTS.md                    # this file
├── CONTRIBUTING.md              # contribution flow
├── LICENSE                      # MIT
└── skills/
    ├── stackbone-coder/         # generate a piece by interview (orchestrator)
    ├── stackbone/               # SDK / build the agent
    ├── stackbone-cli/           # CLI / publish and operate
    └── stackbone-debug/         # diagnostics
```

## Skill format

Each skill is a folder with a `SKILL.md` at the root that follows the [Agent Skills Open Standard](https://agentskills.io/):

```markdown
---
name: stackbone-cli
description: >-
  Use this skill when ...
license: MIT
metadata:
  author: stackbone
  version: '0.1.0'
  organization: Stackbone
  date: May 2026
---

# Body in markdown...
```

The `description:` is the **trigger** — Claude reads it to decide when to load the skill. Make it specific, list concrete user requests (`Trigger on requests like: scaffold an agent, publish a template, run db migrate`), and tell the agent when to delegate (`For X, use the Y skill instead`).

## How the skills compose

| Skill             | Audience inside the agent's day                         | Delegates to                                           |
| ----------------- | ------------------------------------------------------- | ------------------------------------------------------ |
| `stackbone-coder` | Starting a new piece — interviews, then scaffolds it    | `stackbone` for the code, `stackbone-cli` for commands |
| `stackbone`       | Writing eve agents + workflows in a workspace (SDK use) | `stackbone-cli` for build / publish / db migrations    |
| `stackbone-cli`   | Operating the CLI to scaffold, develop and publish      | `stackbone` for SDK code inside the agent              |
| `stackbone-debug` | Triage when something fails                             | `stackbone-cli` for the actual commands                |

If you find yourself repeating content across two skills, prefer cross-linking (`see the stackbone skill for X`) over duplicating — drift is the enemy.

## Documentation patterns

Inside `skills/stackbone/<module>/`:

| File                 | Purpose                                                                    |
| -------------------- | -------------------------------------------------------------------------- |
| `sdk-integration.md` | How to use one `@stackbone/sdk` surface via the ambient `stackbone` client |

Inside `skills/stackbone-cli/references/`:

| File           | Purpose                                                                    |
| -------------- | -------------------------------------------------------------------------- |
| `<command>.md` | Long-form reference for a single CLI command (flags, examples, exit codes) |

## When adding a new module to `stackbone`

1. Create the folder under `skills/stackbone/<module>/`.
2. Write `sdk-integration.md` with: setup, usage examples, best practices, common mistakes (this 4-section shape is load-bearing — Claude scans for it).
3. Update the module reference table in `skills/stackbone/SKILL.md`.

## When adding a new CLI command

1. Add the row in the command table inside `skills/stackbone-cli/SKILL.md`.
2. Add a long-form reference in `skills/stackbone-cli/references/<command>.md` if the command has flags / outputs / edge cases worth documenting.
3. If the command's exit codes diverge from the defaults documented in `SKILL.md`, call it out in the **Non-Obvious Behaviors** section.

## Validation checklist before committing

- [ ] `SKILL.md` frontmatter is valid YAML (`name`, `description`, `license`, `metadata`).
- [ ] Skill `name` matches the directory name (lowercase, hyphens only).
- [ ] `description` is specific enough that Claude can decide when to load it without reading the body.
- [ ] Code examples are syntactically correct and runnable against the published `@stackbone/sdk` and `@stackbone/cli` versions.
- [ ] All internal links are valid relative paths.
- [ ] References between skills use the skill **name** (`use the stackbone-cli skill`), not "the CLI skill" or "the other one".

## Key Stackbone patterns to remember when writing skills

1. **Glossary discipline** — `agent_template` is the marketplace recipe (a row in `stackbone_platform.agent_template`); `agent` is the provisioned instance with its own Neon + R2 + durable execution. **A `workspace` is source code on disk** (what a creator scaffolds with `stackbone init` — eve agents + workflows). Do not blur these.
2. **`{ data, error }` envelope** — every SDK method returns `{ data, error }`. Examples must show both branches.
3. **Env vars are injected** — the creator never hardcodes `DATABASE_URL`, `AWS_ACCESS_KEY_ID`, `OPENROUTER_API_KEY`, etc. The platform injects them at runtime. Saying "set this env var" is wrong — the right framing is "this env var will be available at runtime".
4. **No HTTP code, no Dockerfile** — the runtime serves each eve agent (`/eve/v1/*`) and the workflows (`/api/workflows/*`); the creator only writes agents (`agent.ts` + tools) and workflows (`'use workflow'` / `'use step'`).
5. **Persistence** — relational data, vectors, full-text and KV cache live in the agent's dedicated Neon (`stackbone.database`); durable workflow/step state lives in managed Redis. No separate vector DB or KV store.
6. **CLI is agent-friendly** — `--json`, `--yes`, semantic exit codes, contracted output envelopes. Skills should assume the agent passes `--json --yes` by default.
7. **Tenancy** — organizations have members with roles `owner` / `admin` / `member` / `approver`. The CLI operates against the active organization (resolved from the device-flow session); cross-org commands take `--agent <id>`.
