---
name: stackbone
description: >-
  Use this skill when writing the inside of a Stackbone workspace with @stackbone/sdk:
  the workspace is discovered by convention from the files on disk (deep-agents/<name>/index.ts
  + workflows/<name>.workflow.ts), authoring a deep agent (a single index.ts default-exporting
  defineDeepAgent({ model, systemPrompt, tools, subagents, interruptOn }) from @stackbone/sdk/deep,
  with LangChain tools and connectorTool(...) for third-party operations), writing durable workflows
  as 'use workflow' / 'use step' functions with sibling inputSchema/outputSchema, calling an agent
  from a workflow step with callDeepAgent(name, input) from @stackbone/sdk/workflow, and reaching the
  ambient `stackbone` client from inside a tool or a workflow step — stackbone.database (Drizzle over
  Neon Postgres), stackbone.storage (file uploads), stackbone.ai (LLM chat / embeddings via OpenRouter),
  stackbone.rag (ingest + hybrid search), stackbone.config / stackbone.secrets / stackbone.prompts,
  and stackbone.connection(id) to call a third-party connector — plus human-in-the-loop both ways:
  requestApproval() in a workflow body and tool-level interruptOn pauses inside an agent turn.
  Trigger on requests like: build an agent, add a tool, write a workflow, store user data, upload a file,
  call an LLM, ingest docs for RAG, pause until a human approves, gate a tool behind approval,
  call another agent, call a connector, read dynamic config.
  For CLI tasks (init, dev, publish, db migrate, runs, hitl), use the stackbone-cli skill instead.
  For triage of errors and stuck runs, use the stackbone-debug skill.
license: MIT
metadata:
  author: stackbone
  version: '1.1.0'
  organization: Stackbone
  date: July 2026
---

# Stackbone SDK skill

This skill covers writing code **inside** a Stackbone workspace — what the creator authors with `@stackbone/sdk`. The workspace is discovered by convention from the files on disk (`deep-agents/<name>/index.ts` + `workflows/<name>.workflow.ts`). For everything around the workspace (scaffolding, the dev emulator, publishing, db migrations, triggering runs) use the **stackbone-cli** skill.

## The mental model in four sentences

- A workspace is a project that contains **deep agents** and **workflows**, **discovered by convention from the files on disk**: every `deep-agents/<name>/` folder with an `index.ts` default-exporting `defineDeepAgent(...)` is an agent, and every `workflows/<name>.workflow.ts` is a workflow. An optional `stackbone.config.ts` (default-exporting `defineWorkspace({ agents: [], workflows })`) can override the **workflow** list only — deep agents are always discovered from their folders. Most projects need none.
- A **deep agent** is a [deepagents](https://github.com/langchain-ai/deepagentsjs) (LangGraph) agent: a model + system prompt + tools that hold a conversation across turns. You author it as **one file** — `deep-agents/<name>/index.ts` — with the system prompt inline. It runs **in-process** inside the runtime (no server, no port of its own); the runtime serves it to any client over the standard **OpenAI Chat Completions** and **Anthropic Messages** endpoints, selected by the `model` field. You never write HTTP code.
- A **workflow** is durable, replayable code: a plain async function marked `'use workflow'` whose individual side-effects are marked `'use step'`. Each step is a checkpoint — kill the runtime mid-run and it resumes from the last completed step. Workflows run on the [Workflow SDK](https://workflow-sdk.dev/docs). A workflow step calls a sibling agent in-process with `callDeepAgent(name, input)`.
- All persistence lives in the agent's own Postgres (Neon): relational data, vectors (`pgvector`), full-text (`tsvector`), the approvals inbox, the prompt catalog, dynamic config. From any tool or any workflow step you reach it through the **ambient `stackbone` client** — `import { stackbone } from '@stackbone/sdk'` — with no `createClient()` and no credential wiring.

## Quick setup

### 1. Scaffold (CLI)

Use the **stackbone-cli** skill. `stackbone init` is **workspace-first** — it emits the workspace shell (a `deep-agents/` folder, a `workflows/` folder, `package.json` with the runtime deps pinned, `tsconfig.json`, an `.npmrc`, `.gitignore`, a README) and links it to your org, so you must be signed in. A first piece is opt-in via `--with`: `empty` (shell only — the default), `agent`, `workflow`, or `workflow-agent`. Once you have a workspace, add more pieces **offline** with `stackbone add deep-agent|workflow|workflow-agent <name>` — see the **stackbone-cli** skill for the full command surface.

### 2. Install the SDK (and the peers you use)

```sh
npm install @stackbone/sdk
```

`@stackbone/sdk` declares `deepagents`, `@langchain/core`, `@langchain/langgraph`, `@langchain/openai` and `workflow` as **optional peer dependencies**, resolved from the **workspace root** `package.json` (they must live there — a single copy per process). `stackbone init` / `stackbone add deep-agent` pin them for you; if you wire a workspace by hand:

```sh
npm install deepagents @langchain/core @langchain/langgraph @langchain/openai   # to author a deep agent
npm install workflow                                                            # to author a durable workflow
npm install @langchain/langgraph-checkpoint-postgres                            # durable server-side sessions + tool-level HITL
```

A workflow that never imports `@stackbone/sdk/deep` never loads the LangChain peers, and `import { stackbone } from '@stackbone/sdk'` stays peer-free.

### 3. Declare the workspace (optional, workflows only)

You usually **do not** write this. Deep agents are **always** discovered from their `deep-agents/<name>/index.ts` folders — a `stackbone.config.ts` cannot declare them (its `agents` list is legacy and stays empty). Add one only to **override the workflow scan**:

```ts
// stackbone.config.ts — OPTIONAL override; only the workflow list can be overridden
import { defineWorkspace } from '@stackbone/sdk';

export default defineWorkspace({
  agents: [],
  workflows: [
    {
      name: 'onboarding',
      module: 'workflows/onboarding.workflow.ts',
      export: 'onboardingWorkflow',
    },
  ],
});
```

A **workflow** entry is `{ name, module, export }`. The `name` is how you address/trigger it (`stackbone workflows schema <name>`, `POST /api/workflows/<name>/start`); `module` is the `*.workflow.ts` path; `export` is the exported function name inside it.

## Authoring a deep agent

An agent is **one file**, `deep-agents/<name>/index.ts` — the folder name is the agent's name:

```
deep-agents/support/
  index.ts               ← default export = defineDeepAgent({ model, systemPrompt, tools, ... })
```

```ts
// deep-agents/support/index.ts
import { tool } from '@langchain/core/tools';
import { defineDeepAgent, connectorTool } from '@stackbone/sdk/deep';
import { stackbone, z } from '@stackbone/sdk';

const readConfig = tool(
  async () => {
    const greeting = await stackbone.config.get('greeting');
    return JSON.stringify(greeting.error ? null : greeting.data);
  },
  {
    name: 'read_config',
    description: "Return the agent's current dynamic configuration.",
    schema: z.object({}),
  },
);

export default defineDeepAgent({
  // A bare model id routes through the managed OpenRouter bridge (the org's
  // sub-key is injected at runtime as OPENROUTER_API_KEY). Pass a built
  // LangChain chat-model instance instead for full control.
  model: 'anthropic/claude-haiku-4.5',
  // The system prompt lives INLINE here (or imported from a sibling .ts file
  // for long prompts) — never a .md file loaded by convention.
  systemPrompt: 'You are the support agent. Use your tools before answering.',
  tools: [
    readConfig,
    // A third-party operation as a tool — brokered by Stackbone Connect:
    connectorTool({ connector: 'slack', operation: 'chat.postMessage' }),
  ],
  // Optional: gate a tool behind human approval (see the HITL section below):
  // interruptOn: { send_mail: true },
});
```

- Tools are plain **LangChain tools** (`tool()` from `@langchain/core/tools`, with a Zod `schema`); inside the tool body you reach any ambient surface — `stackbone.database`, `stackbone.ai`, `stackbone.storage`, ….
- `subagents` accepts deepagents sub-agent configs verbatim; `interruptOn` gates tools behind human approval.
- Runtime dependencies (`deepagents`, `@langchain/*`) live in the **workspace root** `package.json` — a deep agent has no per-agent `package.json`.

> **Deep dive:** [agents/authoring.md](agents/authoring.md) — the full `defineDeepAgent` config, the tool pattern, and how the agent is served over the standard wire.

## Authoring a durable workflow

A workflow is a plain async function in `workflows/<name>.workflow.ts`, marked `'use workflow'`, that declares its contract with **sibling `inputSchema` / `outputSchema` exports** (the build harvests them — there is no `defineWorkflow` wrapper). Side-effects live in helper functions marked `'use step'`.

```ts
// workflows/onboarding.workflow.ts
import { z } from '@stackbone/sdk';
import { stackbone } from '@stackbone/sdk';

// THE contract — sibling exports next to the bare function. The build harvests
// these into the manifest + a live validator. The input parameter type is
// DERIVED from inputSchema (z.infer), so there is no second type that can drift.
export const inputSchema = z.object({
  userId: z.string(),
  email: z.email(),
  plan: z.enum(['free', 'pro', 'scale']),
});

// Output is DECLARED (not inferred from the TS return type — that loses fields).
export const outputSchema = z.object({
  userId: z.string(),
  subject: z.string(),
  body: z.string(),
  tips: z.array(z.string()),
});

type OnboardingInput = z.infer<typeof inputSchema>;

export async function onboardingWorkflow(input: OnboardingInput) {
  'use workflow';

  const signup = await validateSignup(input); // step 1 — deterministic
  const tips = await askSupportForTips(signup.plan); // step 2 — calls the support AGENT
  const saved = await persistWelcome(signup.userId, tips); // step 3 — side effect (idempotent)
  return saved;
}

async function validateSignup(input: OnboardingInput) {
  'use step';
  if (!input.email.includes('@')) throw new Error(`Invalid email: ${input.email}`);
  return { userId: input.userId, plan: input.plan };
}

async function persistWelcome(userId: string, tips: { tips: string[] }) {
  'use step'; // deterministic side effect (DB / email). Keep it idempotent on userId.
  await stackbone.database.insert(welcomes).values({ userId, tips: tips.tips });
  return { userId, subject: 'Welcome', body: '...', tips: tips.tips };
}
```

Rules that matter for workflows:

- **`'use workflow'`** marks the durable orchestrator. The body should be cheap, deterministic glue — it replays on resume. Do the I/O in steps.
- **`'use step'`** marks a checkpoint that runs **once**, is persisted, and is retried on failure. Make every step **idempotent** — on retry it may run again.
- **Sibling `inputSchema` / `outputSchema`** are the workflow's public contract. The build extracts them so Studio renders an input form and the emulator validates `start` payloads. Derive the input parameter type with `z.infer<typeof inputSchema>`; declare `outputSchema` explicitly.
- See `/docs/concepts/workflows` (in the wiki) for the durable-execution model in depth.

> **Deep dive:** [workflows/authoring.md](workflows/authoring.md) — the directive rules, the typed contract, pauses, and every trigger path.

## Triggering and scheduling workflows

From inside a workflow, start another workflow **by name** through the `stackbone.workflows` surface on the ambient client (these bind on dispatch, so they run from a workflow body, not a tool):

```ts
import { stackbone } from '@stackbone/sdk';

const { runId } = await stackbone.workflows.start('reconcile', { invoiceId }); // fire-and-forget → its own run
const summary = await stackbone.workflows.startAndWait<Summary>('summarize', { docId }); // durably wait for the output
await stackbone.workflows.schedule('daily-digest', { scope: 'all' }, '0 9 * * *'); // dynamic cron, idempotent by name
```

- `stackbone.workflows.start(name, input)` enqueues an independent run and returns `{ runId }`; `stackbone.workflows.startAndWait(name, input)` suspends durably until it finishes and returns the validated output. `name` is the `*.workflow.ts` convention name (narrowed to the workspace's declared names once `stackbone dev` has generated `.stackbone/workflows.d.ts`); input is validated against the target's schema before anything is enqueued.
- `stackbone.workflows.schedule(name, input, cron)` / `.unschedule(name)` / `.listSchedules()` manage **dynamic** cron triggers (one per `name` — re-scheduling replaces it). For schedules that ship with the workspace, prefer the declarative form — `export const schedules = [{ cron, input }]` next to the workflow — which the build harvests and the runtime reconciles on deploy.
- `ingestDocuments({ collection, content, contentType })` is the durable RAG-ingest trigger (it stages the content, then runs the reserved `rag-ingest` workflow).
- The shell mirror is `stackbone workflows start <name>` (see the **stackbone-cli** skill).

## The ambient `stackbone` client

`import { stackbone } from '@stackbone/sdk'` — the single handle for every agent surface, reachable from any tool `execute()` or workflow step. It resolves config from the injected environment lazily on first use; there is no `createClient()` and you never pass connection strings.

| Surface                    | What it is                                                                                                  | Envelope                                 |
| -------------------------- | ----------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| `stackbone.database`       | **Native Drizzle ORM** + `postgres-js` over the agent's Neon (vectors with `pgvector`, FTS with `tsvector`) | rows / **throws** (no `{ data, error }`) |
| `stackbone.storage`        | File uploads / downloads / signed URLs (R2 in prod, MinIO in dev)                                           | `{ data, error }`                        |
| `stackbone.ai`             | LLM chat / embeddings / vision via OpenRouter (300+ models). Stream long completions.                       | `{ data, error }`                        |
| `stackbone.rag`            | Ingest + hybrid (`pgvector` + `tsvector`) retrieval; tables are platform-provisioned                        | `{ data, error }`                        |
| `stackbone.config`         | Read dynamic config the operator edits in Studio: `get(key)` / `getAll()`                                   | `{ data, error }`                        |
| `stackbone.secrets`        | Read operator-managed encrypted secrets: `get(name)` / `getMany(names)`                                     | `{ data, error }`                        |
| `stackbone.prompts`        | Versioned prompt catalog: `get()` / `compile(key, vars)` / `list()` / `create()` / …                        | `{ data, error }`                        |
| `stackbone.approval`       | Agent-local HITL records (used by `requestApproval` under the hood — see below)                             | `{ data, error }`                        |
| `stackbone.connection(id)` | Call a third-party connector by id (Stackbone Connect — see below)                                          | `Promise<output>` / **throws** by code   |

> To call a **sibling agent** from a workflow step, use `callDeepAgent(name, input)` from `@stackbone/sdk/workflow` (see below) — it is a standalone helper, not a `stackbone.*` surface.

The one rule across every surface: **destructure `{ data, error }` and handle both branches.** The sole exception is `stackbone.database` (native Drizzle — it returns rows and throws).

```ts
const { data, error } = await stackbone.storage
  .from('avatars')
  .upload(file, { contentType: 'image/png' });
if (error) throw new Error(error.message); // propagate the failure
// success: `data` is typed
```

`throw` to propagate a failure; branch on `error.code` to handle a known case and `return` instead. **Never swallow the `error` branch.** The error envelope is `{ code, message, meta? }` (`SdkError`).

> `stackbone.database` is native Drizzle: tables are `pgTable` objects from the workspace's `src/schema.ts`; helpers (`eq`, `and`, `sql`, …) come from `@stackbone/sdk/db`. Awaiting a query returns the typed rows and **throws** on error — there is no envelope.

### Calling a sibling agent — `callDeepAgent`

From a workflow step, call a sibling agent **in-process** with `callDeepAgent(name, input)` from `@stackbone/sdk/workflow` — no HTTP, no session plumbing. The whole agent turn runs as that one durable step.

```ts
import { callDeepAgent } from '@stackbone/sdk/workflow';

async function askSupportForTips(plan: string) {
  'use step';
  const { text } = await callDeepAgent(
    'support',
    `A customer joined the "${plan}" plan. Give up to 3 onboarding tips.`,
  );
  return text;
}
```

`input` is a bare string (one user message) or `{ messages: [{ role, content }] }` (roles `system` | `user` | `assistant`) to carry prior turns; the result is `{ text }`. `name` is the `deep-agents/<name>/` folder basename, narrowed to the workspace's real agent names once `stackbone dev` has generated `.stackbone/agents.d.ts`.

> **Deep dive:** [workflow-agents/authoring.md](workflow-agents/authoring.md) — calling an agent from a workflow, multi-turn input, and idempotency of the turn-as-step.

### Calling a connector — `stackbone.connection(id)`

Call a third-party connector operation directly — no agent, no model in the loop. The credential is brokered by Stackbone Connect; you only pass the operation arguments.

```ts
async function sendMail(input: { to: string; subject: string; body: string }) {
  'use step';
  // Typed from the connector's schema (generated into .stackbone/connect.d.ts on
  // `stackbone dev`). Equivalent to .call('sendMail', input) at runtime.
  const output = await stackbone.connection('stub-mail').sendMail(input);
  return { sent: true, output };
}
```

- `stackbone.connection(id).<operation>(args)` is the typed form (once `.stackbone/connect.d.ts` is generated); `stackbone.connection(id).call('<operation>', args)` is the always-available dynamic escape hatch (use it for a dotted operation id like `'chat.postMessage'`).
- The call returns the operation output and **throws** a `ConnectorCallError` on failure — match `err.code` (the broker taxonomy: `invalid_args`, `credential_error`, `timeout`, …), never `instanceof`.
- See `/docs/concepts/connect` (in the wiki) for the broker model and how operators register connector credentials. To hand a connector operation **to the model as a tool**, use `connectorTool({ connector, operation })` from `@stackbone/sdk/deep` in the agent's `tools` array (see [connections/sdk-integration.md](connections/sdk-integration.md)).

## Human-in-the-loop — two levels

HITL exists at two levels, resolved through the **same approvals inbox** (Studio HITL tab, `stackbone hitl approve|reject`):

- **Tool-level (inside an agent turn)** — `interruptOn: { <tool>: true }` on `defineDeepAgent`. The turn pauses before the gated tool runs, an approvals row appears, and the decision resumes the turn server-side on the same session. Requires a durable session (the client sends `x-stackbone-session`). See [hitl/sdk-integration.md](hitl/sdk-integration.md).
- **Workflow-level (between steps)** — `requestApproval()` from `@stackbone/sdk/workflow` (NOT the main barrel — the subpath statically imports the `workflow` peer). It writes a row the Studio inbox shows, then races the human decision against a timeout:

```ts
// workflows/refund.workflow.ts — pause until a human decides.
// inputSchema / outputSchema are declared as usual (see the workflow example above).
import { requestApproval } from '@stackbone/sdk/workflow';

export async function refundWorkflow(input: z.infer<typeof inputSchema>) {
  'use workflow';

  const decision = await requestApproval({
    token: input.approvalToken, // resume key, unique per approval in a run
    topic: 'refund',
    payload: { orderId: input.orderId, amount: input.amount },
    title: 'Approve refund',
    timeout: '24h', // ISO duration or ms
    fallback: 'reject', // applied if the timeout wins the race
  });

  // Gate the side effect on a fresh approval; decision.timedOut means the fallback fired.
  if (decision.status !== 'approved') return { refunded: false, decision: decision.status };

  await performRefund(input.orderId, input.amount); // a 'use step' — never runs without a fresh decision
  return { refunded: true, decision: decision.status };
}
```

`requestApproval` returns `{ status: 'approved' | 'rejected', payload?, timedOut }`. **It must be called from the workflow body, never inside a `'use step'`** — pausing on a hook is a workflow primitive that suspends the run. The decision-resume path (`stackbone hitl approve/reject`, the Studio inbox) is in the **stackbone-cli** skill. For advanced cases the subpath also re-exports the raw `defineHook` + `sleep` from `workflow` as an escape hatch.

## Typed config — `config.schema.ts`

Declare your dynamic config once at the workspace root in `config.schema.ts` as a Zod object:

```ts
// config.schema.ts
import { z } from 'zod';

export const configSchema = z.object({
  greeting: z.string().min(1).describe('How the agent opens a conversation'),
  maxRetries: z.number().int().min(0).max(10).default(3),
  replyTone: z.enum(['formal', 'casual', 'friendly']).default('friendly'),
  rateLimit: z.object({ perMinute: z.number().int().min(1).default(60) }),
});
```

`stackbone dev` extracts ONE JSON Schema from it — which draws the Config editor form in Studio and validates writes — **and** generates `.stackbone/config.d.ts`, which augments `@stackbone/sdk`'s `ConfigRegistry`. As a result `stackbone.config.get('greeting')` is **key-checked and typed**: a typo is a compile error, `greeting` is `string`, `replyTone` is the enum union, and `rateLimit` is a navigable nested object.

## Injected environment (read by the runtime)

The runtime injects these at boot; **never hardcode them, never ask the user for them, never declare them in `agent.yaml`.** They are platform-managed.

| Category    | Variables                                                                       |
| ----------- | ------------------------------------------------------------------------------- |
| Identity    | `AGENT_ID`, `WORKSPACE_ID`, `STACKBONE_INSTALLATION_ID`                         |
| Persistence | `DATABASE_URL`, `STACKBONE_SECRET_KEY`                                          |
| Security    | `HMAC_SECRET` (signs the runtime's session/workflow/connector calls)            |
| Workflows   | `WORKFLOW_REDIS_URL` (durable step state)                                       |
| LLM         | `OPENROUTER_API_KEY`, `OPENROUTER_BASE_URL` (managed sub-key, passthrough cost) |

> Observability needs **no env var from you**: the runtime auto-instruments outbound calls and correlates `console.*` output with the right run. There is no observability surface to configure.

## agent.yaml — the optional workspace manifest

A deep agent needs **no manifest** — the `deep-agents/<name>/` folder plus its `index.ts` is the whole definition. An `agent.yaml` at the **workspace root** is optional and not scaffolded; when present, `stackbone dev` reads only `database.schema` / `database.migrations` and `dev.autoMigrate` from it (everything else falls back to convention defaults — `./src/schema.ts`, `./.stackbone/migrations`, auto-migrate on). The richer blocks (runtime, rag, connections, automations) belong to the classic single-agent manifest / the publish-time build. The schema is `.strict()` — an unknown key fails parse. See [agent-yaml.md](agent-yaml.md) for the full reference.

## Authoring guides

The full authoring reference for each workspace piece:

| Piece                                                | Doc                                                          |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| A deep agent (model, system prompt, tools)           | [agents/authoring.md](agents/authoring.md)                   |
| A durable workflow (`'use workflow'` / `'use step'`) | [workflows/authoring.md](workflows/authoring.md)             |
| A workflow that calls an agent (`callDeepAgent`)     | [workflow-agents/authoring.md](workflow-agents/authoring.md) |

## Per-surface deep dives

The full method shapes and worked examples for each ambient surface live in the leaf docs — read the one your task touches:

| Task                                                       | Doc                                                              |
| ---------------------------------------------------------- | ---------------------------------------------------------------- |
| Drizzle queries, transactions, vectors, full-text, paging  | [database/sdk-integration.md](database/sdk-integration.md)       |
| Uploads, the `key` handle, public/signed URLs              | [storage/sdk-integration.md](storage/sdk-integration.md)         |
| Chat/embeddings, **streaming** long completions, vision    | [ai/sdk-integration.md](ai/sdk-integration.md)                   |
| Ingest and hybrid retrieval                                | [rag/sdk-integration.md](rag/sdk-integration.md)                 |
| HITL pauses (tool-level `interruptOn` + `requestApproval`) | [hitl/sdk-integration.md](hitl/sdk-integration.md)               |
| Versioned prompt catalog + `compile()`                     | [prompts/sdk-integration.md](prompts/sdk-integration.md)         |
| Connector operations (typed + `.call`)                     | [connections/sdk-integration.md](connections/sdk-integration.md) |
| Triggering & scheduling workflows, background work         | [scheduling/sdk-integration.md](scheduling/sdk-integration.md)   |

## Branch the backend for risky changes

If a change depends on a schema migration, a new RLS policy, a `rag.embeddingModel` change, or anything else that could leave a deployed agent broken — work on a `stackbone dev` session first (your local install, isolated Postgres + Redis + MinIO) before touching the cloud. See the **stackbone-cli** skill (`dev` reference) for the loop.

## Important notes

- **No HTTP framework choice, no Dockerfile.** The runtime serves each agent over the standard **OpenAI Chat Completions** + **Anthropic Messages** endpoints (selected by the `model` field) and the workflows over `/api/workflows/*`; you write neither HTTP routes nor a Dockerfile.
- **One copy of the SDK and LangChain per process.** `stackbone publish` bundles each agent with `@stackbone/sdk`, `deepagents` and `@langchain/*` kept **external**, resolved from the workspace root `node_modules` — an inlined second SDK copy splits the invocation context and per-run logs lose their run id. Keep those deps pinned at the workspace root (the scaffold does this), never in a per-agent `package.json`.
- **Steps must be idempotent.** A `'use step'` is retried on failure and replayed on resume — code it so running twice is safe.
- **The agent owns its data.** Stackbone never reads or writes the agent's Neon directly.
- **`requestApproval` lives on `@stackbone/sdk/workflow`**, not the main barrel — importing it from `@stackbone/sdk` will not resolve.
