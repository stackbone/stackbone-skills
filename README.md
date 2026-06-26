![Stackbone Agent Skills](./github_skills_banner.png)

# Stackbone Agent Skills

[![skills.sh](https://skills.sh/b/stackbone/stackbone-skills)](https://skills.sh/stackbone/stackbone-skills)

Agent Skills for coding agents (Claude Code, Cursor, Windsurf, Cline, Codex, ...) building on [Stackbone](https://stackbone.ai) — the App Store + hosting platform for AI agents.

## What is Stackbone?

Creators author a **workspace** of durable [eve](https://eve.dev) agents and [Workflow SDK](https://workflow-sdk.dev) workflows on `@stackbone/sdk`; organizations install them and get an **agent** that runs in their own cloud with its own database (with vector search), object storage, durable execution (Redis), a human-in-the-loop inbox and an LLM gateway (OpenRouter) baked in. The creator never writes auth, billing, metrics, HTTP routes or integration clients — every step of the lifecycle (scaffold, develop, publish, operate) runs through the `stackbone` CLI.

## Installation

### Using the skills registry

```bash
npx skills add stackbone/stackbone-skills
```

Add `-a claude-code` / `-a cursor` / `-a windsurf` / etc. to target specific agents. With no `-a` flag the registry installs into every supported agent it detects.

### Claude Code marketplace

```
/plugin marketplace add stackbone/stackbone-skills
/plugin install stackbone
```

### Auto-installed by the Stackbone CLI

When you run `stackbone init` or `stackbone link`, the CLI also installs these skills for every supported coding agent and adds the agent directories to `.gitignore`. You can re-run either command at any time to refresh the skills.

## Available skills

<details>
<summary><strong>stackbone-coder</strong> — Generate a piece by interview (start here)</summary>

Turn _"I want to build X"_ into a scaffolded, wired-up Stackbone piece through a guided interview. Covers:

- **Pick the shape** — decide between an **agent** (conversational, has tools), a **workflow** (durable `input → output` pipeline) or a **workflow-agent** (a workflow that calls an agent), then scaffold it with `stackbone init --with` / `stackbone add`.
- **Type-specific interview** — for an agent, its **tools** and **system prompt**; for a workflow, its **input** and **output** data → the `inputSchema` / `outputSchema`.
- **Capability checklist** — walk every surface one at a time (database, storage, AI, RAG, HITL, connections, prompts, config, secrets, scheduling, calling another agent) and wire in only the ones the user needs.
- **Four reference files** — `agent.md`, `workflow.md`, `workflow-agent.md`, and the full `capabilities.md` checklist.

**Use this skill when**: starting a new agent/workflow from scratch. It orchestrates the interview and scaffolding and delegates the code to `stackbone` and the commands to `stackbone-cli`.

</details>

<details>
<summary><strong>stackbone</strong> — SDK / build the agent</summary>

Write the inside of a workspace — eve agents and durable workflows — using `@stackbone/sdk` and the ambient `stackbone` client. Covers:

- **Agents & workflows** — author a durable eve agent (`agent.ts` + `instructions.md` + tools) and durable workflows (`'use workflow'` / `'use step'` with sibling `inputSchema` / `outputSchema`); reach every surface through the ambient `stackbone` client from a tool's `execute()` or a workflow step.
- **Database** — Drizzle ORM over Neon Postgres, `pgvector`, `tsvector`, transactions (`stackbone.database` — native Drizzle, throws).
- **Storage** — Cloudflare R2 via `@aws-sdk/client-s3` (MinIO in dev), logical buckets (`stackbone.storage`).
- **AI** — OpenRouter through the official `openai` SDK shape, passthrough billing (`stackbone.ai`).
- **RAG** — parser + chunker + embeddings + `pgvector` retrieval; schema platform-provisioned (`stackbone.rag`).
- **Triggering & scheduling** — start workflows by name and register cron schedules (`stackbone.workflows.start` / `stackbone.workflows.schedule`).
- **HITL** — durable pauses with `requestApproval()` from `@stackbone/sdk/workflow`; decide in Studio or the CLI.
- **Secrets / config / prompts / connections** — agent-local primitives (`stackbone.secrets`, `stackbone.config`, the versioned `stackbone.prompts` catalog, `stackbone.connection(id)` for third-party connectors).
- **agent.yaml manifest** — the per-agent recipe (apiVersion, name, runtime, database, dev, rag.embeddingModel, connections, automations, protocol — a `.strict()` schema). No Dockerfile, no HTTP routes — the runtime serves the agent + workflows.

**Use this skill when**: writing agent/workflow logic, wiring a `stackbone` surface, declaring `agent.yaml`. For CLI tasks (scaffold, publish, db migrations), use `stackbone-cli` instead.

</details>

<details>
<summary><strong>stackbone-cli</strong> — CLI / publish and operate</summary>

Drive the `stackbone` CLI to scaffold, develop, publish and operate a workspace. Covers:

- **Auth** — `stackbone login` (device flow, RFC 8628), `whoami`, `current`.
- **Project lifecycle** — `init` (scaffold a workspace shell, offline by default), `add agent|workflow|workflow-agent` (grow it), `link`, `organization use`, `metadata`.
- **Local stack** — `stackbone dev` boots the whole workspace (Postgres, Redis, MinIO) with Studio at `:4242` and hot reload.
- **Workflows** — `stackbone workflows list / schema / start` to inspect and trigger the durable workflows an install exposes.
- **Publish** — `stackbone publish` compiles every agent + workflow and packs the workspace bundle tar (digest-verifiable; the platform provisioning flow uploads it).
- **Database** — `stackbone db migrate up/create/status` plus a read-only `db query / schemas / table` explorer; the RAG schema is platform-provisioned.
- **Operate an install** — runs, logs, hitl, secrets, config, storage, rag, prompts, contract, openrouter (local-dev install by default, or `--agent <id>`).
- **Docs in the terminal** — `stackbone docs` prints SDK / CLI / agent.yaml docs inline.
- **Agent-friendly defaults** — `--json`, `--yes`, `--verbose`; semantic exit codes (`0` ok, `3` no project linked, etc.).

**Use this skill when**: the user asks to create, build, publish or operate an agent via the CLI. For SDK code inside the agent, use `stackbone` instead.

</details>

<details>
<summary><strong>stackbone-debug</strong> — Diagnostics</summary>

Triage failures in a Stackbone workspace (`stackbone dev` or a deployed install). Covers:

- **HITL runs stuck** — `stackbone hitl list`, decide / inspect a parked `requestApproval` gate.
- **Durable runs failing or timing out** — read the run timeline + per-step log frames.
- **Workflow input rejected** — a `start`/`chat` body that fails the workflow's `inputSchema` (400 `workflow_input_invalid`).
- **SDK errors** — the `{ data, error }` envelope and the most common `<prefix>_<reason>` codes.
- **HTTP 4xx/5xx from the control plane** — auth, RBAC capability mismatches, tier quota (402).
- **Database** — slow queries, missing indexes, `pgvector` distance vs index mismatch.
- **Secrets / config** — decryption errors, missing injected env vars.
- **Scheduled workflows** — a cron schedule that never fired, fired twice, or whose run failed.
- **Publish / build** — `stackbone publish` aborts (native dep, `@stackbone/sdk` inlined instead of external, eve partially traced).

**Use this skill when**: the user reports an error, a stuck run, an unexpected HTTP status, or asks "why didn't this work". The skill guides diagnostic command execution; it does not propose fixes.

</details>

## Skill structure

Each skill follows the [Agent Skills Open Standard](https://agentskills.io/):

```
skills/
├── stackbone-coder/
│   ├── SKILL.md                 # the interview orchestrator (start here)
│   └── references/
│       ├── agent.md
│       ├── workflow.md
│       ├── workflow-agent.md
│       └── capabilities.md
├── stackbone/
│   ├── SKILL.md                 # frontmatter + body, the entrypoint Claude reads
│   ├── database/sdk-integration.md
│   ├── storage/sdk-integration.md
│   ├── ai/sdk-integration.md
│   ├── rag/sdk-integration.md
│   ├── connections/sdk-integration.md
│   ├── prompts/sdk-integration.md
│   ├── scheduling/sdk-integration.md
│   ├── hitl/sdk-integration.md
│   └── agent-yaml.md
├── stackbone-cli/
│   ├── SKILL.md
│   └── references/
│       ├── init.md
│       ├── login.md
│       ├── dev.md
│       ├── publish.md
│       └── db.md
└── stackbone-debug/
    └── SKILL.md
```

The frontmatter `description:` field is the trigger — Claude reads it to decide when to load a skill. Skills reference each other by name (`use the stackbone-cli skill instead`) so the agent routes itself.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Skills are Markdown files with YAML frontmatter — easy to edit, easy to PR.

## License

[MIT](LICENSE)
