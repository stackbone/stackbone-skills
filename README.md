![Stackbone Agent Skills](./github_skills_banner.png)

# Stackbone Agent Skills

[![skills.sh](https://skills.sh/b/stackbone/stackbone-skills)](https://skills.sh/stackbone/stackbone-skills)

Agent Skills for coding agents (Claude Code, Cursor, Windsurf, Cline, Codex, ...) building on [Stackbone](https://stackbone.ai) — the App Store + hosting platform for AI agents.

## What is Stackbone?

Creators publish containerized **agent templates** built on `@stackbone/sdk` and a language-agnostic Stackbone runtime; organizations install them and get an **agent** (instance) with its own database (with vector search), object storage, durable queues, HITL inbox and LLM gateway baked in. The creator never writes auth, billing, metrics or integration clients — every step of the lifecycle (scaffold, develop, publish, operate) runs through the `stackbone` CLI.

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
<summary><strong>stackbone</strong> — SDK / build the agent</summary>

Write the inside of an agent template using `@stackbone/sdk`. Covers:

- **Database** — Drizzle ORM over Neon Postgres, `pgvector`, `tsvector`, KV cache with `expires_at`, the `_queue_jobs` queue table.
- **Storage** — Cloudflare R2 via `@aws-sdk/client-s3` (MinIO in dev), object metadata in the agent's Neon.
- **AI** — OpenRouter through the official `openai` SDK (base URL override). LLM tokens billed passthrough, no markup.
- **RAG** — LlamaParse (opt-in) + chunker + embeddings via OpenRouter + `pgvector` + `tsvector` hybrid search.
- **Queues** — Upstash QStash publisher + Hono receiver with HMAC; durable cross-container push and scheduled jobs.
- **HITL** — `client.approval` to pause runs, wait for human decision in Studio, and resume.
- **Secrets / config / connections / events** — Stackbone-managed primitives the agent reads at runtime.
- **agent.yaml manifest** — the recipe published as `agent_template` (name, runtime, pricing, capabilities).
- **HTTP contract** — `/invoke`, `/health`, `/schema`. The platform-managed wrapper mounts these; the creator never writes HTTP code, only `defineAgent({ invoke })` in `src/index.ts`.

**Use this skill when**: writing the agent's logic, wiring an SDK module into a workflow, declaring `agent.yaml`. For CLI tasks (build, publish, db migrations), use `stackbone-cli` instead.

</details>

<details>
<summary><strong>stackbone-cli</strong> — CLI / publish and operate</summary>

Drive the `stackbone` CLI to scaffold, develop, publish and operate agents. Covers:

- **Auth** — `stackbone login` (device flow, RFC 8628), `whoami`, `current`.
- **Project lifecycle** — `init` (clone a starter), `link` (attach to an existing template), `metadata` (machine-readable workspace state).
- **Local emulator** — `stackbone dev` runs the control plane locally with Studio at `:4242`, hot reload, dry-run billing.
- **Runtime serve** — `stackbone serve` mounts the agent against the canonical `/invoke` + `/health` + `/schema` contract.
- **Publish** — `stackbone publish` validates, builds the container via the platform buildpack, signs with cosign, pushes to `registry.fly.io` and registers the version.
- **Database** — `stackbone db migrate up/create/status`, `stackbone db studio`, `stackbone db add-rag` (declarative migrations driven by Drizzle).
- **Docs in the terminal** — `stackbone docs` prints SDK / CLI / agent.yaml docs inline, no browser fetch.
- **Agent-friendly defaults** — `--json` for structured output, `--yes` to skip confirmations, `--verbose` to stream logs. Semantic exit codes (`0` ok, `3` no project linked, etc.).

**Use this skill when**: the user asks to create, build, publish or operate an agent template via the CLI. For SDK code inside the agent, use `stackbone` instead.

</details>

<details>
<summary><strong>stackbone-debug</strong> — Diagnostics</summary>

Triage failures in a Stackbone agent (`stackbone dev` or in production). Covers:

- **HITL runs stuck** — `stackbone hitl list`, decide / cancel / inspect payload.
- **Runs failing or timing out** — read run timeline, retrieve logs and traces.
- **SDK errors** — `{ data, error }` envelope and the most common error codes.
- **HTTP 4xx/5xx from the control plane** — auth, RBAC capability mismatches, tier quota exceeded.
- **Database** — slow queries, missing indexes, `pgvector` distance vs index mismatch, RLS / connection issues.
- **Secrets / config** — `installation_secret` decryption errors, missing env vars at runtime.
- **Queues** — QStash signature verification failures, dead-letter inspection.
- **Deployments** — `stackbone publish` failed (Trivy CVE block, cosign signing, registry push).

**Use this skill when**: the user reports an error, an agent stuck, an unexpected HTTP status, or asks "why didn't this work". The skill guides diagnostic command execution; it does not propose fixes.

</details>

## Skill structure

Each skill follows the [Agent Skills Open Standard](https://agentskills.io/):

```
skills/
├── stackbone/
│   ├── SKILL.md                 # frontmatter + body, the entrypoint Claude reads
│   ├── database/sdk-integration.md
│   ├── storage/sdk-integration.md
│   ├── ai/sdk-integration.md
│   ├── rag/sdk-integration.md
│   ├── queues/sdk-integration.md
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
