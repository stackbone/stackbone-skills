![Stackbone Agent Skills](./github_skills_banner.png)

# Stackbone Agent Skills

[![skills.sh](https://skills.sh/b/stackbone/stackbone-skills)](https://skills.sh/stackbone/stackbone-skills)

Agent Skills for coding agents (Claude Code, Cursor, Windsurf, Cline, Codex, ...) building on [Stackbone](https://stackbone.ai) — the App Store + hosting platform for AI agents.

## What is Stackbone?

Creators author a **workspace** of **deep agents** ([deepagents](https://github.com/langchain-ai/deepagentsjs)/LangGraph, served over the standard OpenAI/Anthropic chat endpoints) and [Workflow SDK](https://workflow-sdk.dev) workflows on `@stackbone/sdk`; organizations run them as **agents** in their own cloud with a Postgres (with vector search), object storage, durable execution (Redis), a human-in-the-loop inbox and a model gateway baked in. The creator never writes auth, billing, metrics, HTTP routes or integration clients — every step of the lifecycle (scaffold, develop, build, operate) runs through the `stackbone` CLI.

## How the skills relate to the docs

The skills teach a coding agent **how to work** on Stackbone: the shapes to write, the order of operations, the rules that hold everywhere, and which docs page to read before touching a surface. They hold no API reference. Every flag, method, option and error code lives in the public docs at [docs.stackbone.ai](https://docs.stackbone.ai), which the same docs site serves to the agent over MCP (`https://docs.stackbone.ai/mcp`: `search_docs`, `get_doc`, `list_docs`). `stackbone init` and `stackbone link` install the skills **and** wire that MCP server into the coding agents you pick. Without an MCP client, the agent reads `https://docs.stackbone.ai/llms.txt` and the raw markdown of each page.

## Installation

### Using the skills registry

```bash
npx skills add stackbone/stackbone-skills
```

### Claude Code marketplace

```
/plugin marketplace add stackbone/stackbone-skills
/plugin install stackbone
```

The plugin carries the skills **and** the `stackbone-docs` MCP server (`https://docs.stackbone.ai/mcp`), declared in `.claude-plugin/plugin.json`, so a Claude Code that installs the plugin can read the docs without any other setup.

### Auto-installed by the Stackbone CLI

When you run `stackbone init` or `stackbone link`, the CLI asks which coding agents you use, installs these skills for them, adds the `stackbone-docs` MCP server to each one's configuration, and gitignores the per-agent skill copies. Re-run either command at any time to refresh the skills.

## Available skills

<details>
<summary><strong>stackbone-coder</strong> — Generate a piece by interview (start here)</summary>

Turn _"I want to build X"_ into a scaffolded, wired-up Stackbone piece through a guided interview:

- **Pick the shape** — an **agent** (conversational, has tools), a **workflow** (durable `input → output` pipeline) or a **workflow-agent** (a workflow that calls an agent), then scaffold it with `stackbone init --with` / `stackbone add`.
- **Type-specific interview** — for an agent, its **tools** and **system prompt**; for a workflow, its **input** and **output** data → the `inputSchema` / `outputSchema`.
- **Capability checklist** — walk every surface one at a time (database, storage, AI, RAG, HITL, connections, prompts, config, secrets, settings, scheduling, calling another agent, a browser) and wire in only the ones the user needs; each row names the docs page to read.

**Use this skill when**: starting a new agent/workflow from scratch. It orchestrates the interview and scaffolding and delegates the code to `stackbone` and the commands to `stackbone-cli`.

</details>

<details>
<summary><strong>stackbone</strong> — SDK / write the code</summary>

How to write the inside of a workspace with `@stackbone/sdk`:

- **The three shapes** — a deep agent (`deep-agents/<name>/index.ts` default-exporting `defineDeepAgent` with an inline system prompt and LangChain tools), a durable workflow (`'use workflow'` / `'use step'` with sibling `inputSchema` / `outputSchema`), and a workflow that calls an agent (`callDeepAgent` / `streamDeepAgent` in one step).
- **The rules** — the `{ data, error }` envelope and the three surfaces that throw instead, idempotent steps, `requestApproval()` in the body, peers at the workspace root, injected env never asked for.
- **How to find the page** — a surface's page is titled like the surface, each shape has an `Overview` and a `Getting started`, anything else is a `search_docs`; the agent reads the page before writing.

**Use this skill when**: writing agent/workflow logic or wiring a `stackbone.*` surface. For CLI tasks use `stackbone-cli`.

</details>

<details>
<summary><strong>stackbone-cli</strong> — CLI / scaffold, run, ship, operate</summary>

How to drive the `stackbone` CLI:

- **How to call it** — `--json` on every call, `--yes` on destructive verbs, the exit codes to branch on, what to do when a command fails.
- **Session start** — `whoami` / `metadata` and what each exit code tells you to run next.
- **The loops** — new workspace (`login` → `init` → `dev`), grow it (`add`), the dev loop (`workflows start` → `runs get` → `logs tail`), schema changes (`db migrate`), operating an installation (`--agent`, `hitl`), shipping (`package` / `build` → deploy → `link`), CI.
- **How to find the page** — one reference page per command, titled like it; `Command conventions` for what every command shares; `Troubleshooting` for a boot or targeting failure.

**Use this skill when**: the user asks to scaffold, run, ship or operate a workspace via the CLI. For SDK code use `stackbone`.

</details>

<details>
<summary><strong>stackbone-debug</strong> — Locate the cause</summary>

How to locate a failure in a Stackbone workspace (`stackbone dev` or a deployed installation):

- **The loop** — capture the symptom verbatim, get a code (`--json`, `result.error.code`, the HTTP body, the run frames), run the first read-only diagnostic, read the owning docs page, report and hand off.
- **Symptom → first diagnostic** — failed or stuck runs, a rejected workflow input, parked approvals at both levels, lost conversation memory, schedules, connectors, storage, database, build aborts, model calls.
- **How to find the owner page** — `search_docs` the code verbatim; the prefix names the surface; where CLI codes, HTTP statuses and 402 bodies are explained.

**Use this skill when**: the user reports an error, a stuck run, an unexpected HTTP status, or asks "why didn't this work". It locates the cause and leaves the fix to `stackbone` / `stackbone-cli`.

</details>

<details>
<summary><strong>stackbone-integrations</strong> — Wire the workspace to the outside</summary>

Connections, triggers and destinations as one task, because they are one:

- **Inbound, in order** — register a connector, connect the account it acts as, link one provider event to one workflow, map the event onto the workflow's input, and only then turn it on.
- **Outbound** — the workflow declares what it emits in code; an operator binds each name to a connector, an account and a folder on the box.
- **The traps** — an armed link with no saved mapping swallows the first arrival, a blank mapping canvas overwrites a working mapping, and a connector's scopes freeze when it is registered.

**Use this skill when**: the agent should receive on its own, or what it writes has to land somewhere.

</details>

<details>
<summary><strong>stackbone-guardrails</strong> — Put a limit on a running agent</summary>

A guardrail is box state, not code: no `stackbone.guardrails`, no CLI verb, and an edit takes effect on the next turn.

- **The five nouns** — the rule, the check, the side (`input` / `output`), the action (block, redact, hold) and the attachment.
- **Zero to a rule that fires** — decide the unit and the side, read the box's catalog, create it, and attach it, which is the step everyone skips.
- **Why a rule is not firing** — six silent reasons, walked in order.

**Use this skill when**: the agent has to stop saying something, mask something, or wait for a person.

</details>

<details>
<summary><strong>stackbone-evals</strong> — Measure whether it works</summary>

Turn a manual Playground poke into a suite that answers "did this change help":

- **The surfaces** — the CLI cannot build an eval; it launches one somebody else wrote and turns the verdict into an exit code.
- **Zero to a comparison** — settings, the box's criterion catalog, cases, the suite, a smoke run, then two variants.
- **The vocabulary** — scorer against judge, dataset version, variant, persona simulator, trajectory.

**Use this skill when**: you need to know whether the agent still works, or whether a change made it better or worse.

</details>

## Skill structure

Each skill follows the [Agent Skills Open Standard](https://agentskills.io/):

```
skills/
├── stackbone-coder/
│   ├── SKILL.md                 # the interview orchestrator (start here)
│   └── references/
│       ├── agent.md             # the agent interview
│       ├── workflow.md          # the workflow interview
│       ├── workflow-agent.md    # the hybrid interview
│       └── capabilities.md      # the capability checklist
├── stackbone/
│   └── SKILL.md                 # shapes + rules + how to find the page
├── stackbone-cli/
│   └── SKILL.md                 # how to call it + the loops + how to find the page
├── stackbone-debug/
│   └── SKILL.md                 # the loop + symptom table + how to find the owner page
├── stackbone-integrations/
│   └── SKILL.md                 # connections, triggers and destinations, in order
├── stackbone-guardrails/
│   └── SKILL.md                 # the five nouns + why a rule is not firing
└── stackbone-evals/
    └── SKILL.md                 # what each surface can do + zero to a comparison
```

Every `SKILL.md` opens with the same **Where the facts live** block (the MCP server, `list_docs` at session start, when to `search_docs`, the `llms.txt` fallback). The four that route across the whole product also close with **How to find the page**: the rules that lead from a task to a docs page (a surface's page is titled like the surface, a command's page like the command, the rest is a search). No paths and no catalogue of pages: the agent resolves both through the MCP at run time, so a reorganisation of the docs site never breaks an installed skill. The frontmatter `description:` field is the trigger — Claude reads it to decide when to load a skill. Skills reference each other by name (`use the stackbone-cli skill instead`) so the agent routes itself.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Skills are Markdown files with YAML frontmatter — easy to edit, easy to PR.

## License

[MIT](LICENSE)
