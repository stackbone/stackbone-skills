# Agent guidance — Stackbone Skills repo

This document helps coding agents work effectively when **modifying** the skills in this repository (not when consuming them — for that, the skills speak for themselves).

## Repository structure

```
stackbone-skills/
├── .claude-plugin/plugin.json   # marketplace manifest for Claude Code (the version + the stackbone-docs MCP server)
├── README.md                    # human-facing overview
├── AGENTS.md                    # this file
├── CONTRIBUTING.md              # contribution flow
├── LICENSE                      # MIT
└── skills/
    ├── stackbone-coder/         # generate a piece by interview (orchestrator) + 4 interview scripts
    ├── stackbone/               # SDK: the three shapes, the rules, how to find the page
    ├── stackbone-cli/           # CLI: how to call it, the loops, how to find the page
    ├── stackbone-debug/         # how to locate a cause: symptom → first command, code → owner page
    ├── stackbone-integrations/  # connections, triggers, destinations: wiring the box to the outside
    ├── stackbone-guardrails/    # limits on a running agent: block, redact, hold
    └── stackbone-evals/         # cases, suites, runs: whether it works and whether a change helped
```

## The one rule: skills hold procedure, the docs hold facts

A skill tells a coding agent **how to act**: the order of operations, the shapes to write, the rules that hold everywhere, the pitfalls, and **which docs page to read** before touching a surface. A skill never holds a flag table, an SDK signature list, an `agent.yaml` key reference or an error catalog. Those live in the public docs at `https://docs.stackbone.ai`, served over MCP as the `stackbone-docs` server (`search_docs`, `get_doc`, `list_docs`) and as plain text at `https://docs.stackbone.ai/llms.txt`.

A skill does not carry a catalogue of pages either: `list_docs` is the catalogue. It carries the **rules for finding the page** (a surface's page is titled like the surface; a command's page is titled like the command; anything else is a `search_docs` on the identifier) and names a page by its **title and place** (`` `Troubleshooting` (CLI › Guides) ``) only when the rule alone would not find it. Never a path: the agent resolves paths through `list_docs` at run time. Titles survive a reorganisation of the site; paths do not, and a skill is a snapshot.

Why: the skills are a snapshot installed into a project (`npx skills add`, or the Stackbone CLI on `init` / `link`); the docs are live. A fact copied into a skill drifts the day the CLI or the SDK ships. A pointer does not.

So when you are about to add a table of flags or methods to a skill, stop: fix the docs page instead, and make sure the skill's finding rules reach it. A table of pages is the same mistake one level up.

Every `SKILL.md` carries the same three blocks, in this order, after the frontmatter:

1. **Where the facts live**: the MCP server, `list_docs` at session start, when to `search_docs`, and the `llms.txt` fallback. Keep it self-contained: a skill is loaded on its own.
2. **How to work** (or **How to use this skill**): the numbered procedure.
3. **How to find the page**: the finding rules, as a short bullet list. No table of pages. Any page named by title must exist under that title in the wiki (`list_docs` is the check).

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
  version: '2.0.0'
  organization: Stackbone
  date: August 2026
---

# Body in markdown...
```

The `description:` is the **trigger** — Claude reads it to decide when to load the skill. Make it specific, list concrete user requests (`Trigger on requests like: scaffold a workspace, add an agent, run db migrate`), and tell the agent when to delegate (`For X, use the Y skill instead`).

## How the skills compose

| Skill                    | Audience inside the agent's day                                    | Delegates to                                                   |
| ------------------------ | ------------------------------------------------------------------ | -------------------------------------------------------------- |
| `stackbone-coder`        | Starting a new piece — interviews, then scaffolds it               | `stackbone` for the code, `stackbone-cli` for commands         |
| `stackbone`              | Writing deep agents + workflows in a workspace (SDK use)           | `stackbone-cli` for dev / build / db migrations                |
| `stackbone-cli`          | Driving the CLI to scaffold, run, ship and operate                 | `stackbone` for SDK code inside the workspace                  |
| `stackbone-debug`        | Locating the cause when something fails                            | `stackbone` or `stackbone-cli` for the fix                     |
| `stackbone-integrations` | Wiring the box to the outside: connections, triggers, destinations | `stackbone` for the workflow's own code                        |
| `stackbone-guardrails`   | Limiting a running agent: block, redact, hold                      | `stackbone-cli` for `hitl`, `stackbone` for `interruptOn`      |
| `stackbone-evals`        | Measuring whether it works, and whether a change helped            | `stackbone` for a judge workflow, `stackbone-cli` for the gate |

All seven send the agent to the docs MCP server for any fact. If you find yourself repeating content across two skills, cross-link (`see the stackbone skill for X`) instead of duplicating; if you find yourself repeating a docs page, delete the copy and point.

## When the CLI or the SDK changes

1. Does the change alter **how to act** (a new loop, a retired command, a rule that moved)? Edit the procedure in the affected `SKILL.md`. Retired commands get one line in the skill's rules ("`publish` does not exist; use `build` / `package` + `link`") so an agent that read old material is corrected.
2. Does it add or rename a **surface or a command**? Usually nothing to do: the finding rules reach a new `stackbone.<surface>` or `stackbone <command>` page by name. Edit a skill only when the new page breaks the rule (a page not titled like its surface), and make sure the docs page exists first (the wiki is maintained in the Stackbone monorepo under `apps/wiki`).
3. Does it only change a **flag, a signature, an option or an error code**? Nothing to do here: that is a docs change.
4. Bump the version in `.claude-plugin/plugin.json` and in every `SKILL.md` you touched (same number everywhere). A procedure change is a minor; a structural change (files added or removed) is a major; a wording fix is a patch.

## Validation checklist before committing

- [ ] `SKILL.md` frontmatter is valid YAML (`name`, `description`, `license`, `metadata`).
- [ ] Skill `name` matches the directory name (lowercase, hyphens only).
- [ ] `description` is specific enough that Claude can decide when to load it without reading the body.
- [ ] Every page a skill names exists under that title and section in `list_docs` on `https://docs.stackbone.ai/mcp`. No `/docs/…` path and no table of pages anywhere in a skill.
- [ ] No flag table, SDK signature list, error catalog or page catalogue crept in. Code blocks are the minimal shapes, not worked examples (those live under Examples in the docs).
- [ ] Code examples are syntactically correct against the published `@stackbone/sdk` and `@stackbone/cli` versions.
- [ ] References between skills use the skill **name** (`use the stackbone-cli skill`), not "the CLI skill" or "the other one".
- [ ] Version bumped in `plugin.json` and in each touched `SKILL.md`.

## Key Stackbone patterns to remember when writing skills

1. **Glossary discipline** — a **workspace** is source code on disk (what a creator scaffolds with `stackbone init`: deep agents + workflows); an **agent** is the deployed instance (an installation in an organization) with its own Postgres + object storage + durable execution. Do not blur these.
2. **`{ data, error }` envelope** — every ambient surface returns it, except `stackbone.database` (native Drizzle, throws), `callDeepAgent` / `streamDeepAgent` (throw) and `stackbone.connection(id)` (throws a `ConnectorCallError`). Examples show both branches.
3. **Env vars are injected** — the creator never hardcodes `STACKBONE_POSTGRES_URL`, `MODEL_PROVIDER_API_KEY`, storage credentials, etc. The runtime injects them. The right framing is "this env var is available at runtime", never "set this env var".
4. **No HTTP code, no Dockerfile** — the runtime serves each deep agent over the standard OpenAI / Anthropic chat endpoints (selected by the `model` field) and the workflows over `/api/workflows/*`; the creator writes only `deep-agents/<name>/index.ts` and `workflows/<name>.workflow.ts`.
5. **Persistence** — relational data, vectors and full-text live in the agent's own Postgres (`stackbone.database`); durable workflow state lives in Redis. No separate vector DB.
6. **The CLI is agent-friendly** — `--json`, `--yes`, semantic exit codes, one output envelope. Skills assume the agent passes `--json --yes`.
7. **No publish command** — the creator runs the agent in their own cloud: `stackbone package` or `stackbone build`, deploy the image, `stackbone link --installation --url --secret --tag` (one box per installation; `link` never creates one).
