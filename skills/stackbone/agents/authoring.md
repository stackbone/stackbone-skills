# Authoring an eve agent

An **agent** is a durable [eve](https://eve.dev/docs/introduction) agent: a model + instructions + tools that hold a conversation across turns. One folder under `agents/<name>/` is one deployable agent. Scaffold it with `stackbone add agent <name>` (see the **stackbone-cli** skill); this page is the reference for what those files contain.

## Layout

```
agents/<name>/
  package.json            ← name: "<name>"; deps: eve, @stackbone/sdk, @ai-sdk/openai-compatible, zod
  agent.yaml              ← the per-agent manifest (see agent-yaml.md)
  agent/
    agent.ts              ← model + build config (default export = eve's defineAgent)
    instructions.md       ← the system prompt (eve loads it by convention)
    tools/<tool>.ts       ← one tool per file (default export = defineTool)
    channels/ , auth/     ← scaffolded HMAC infra you don't normally touch
  schema.ts               ← optional Drizzle tables (path set by agent.yaml database.schema)
```

The agent's name comes from `agent.yaml`'s `name:` and **must equal the folder basename** — the convention scan keys agents on it. You do not write the name in `agent.ts`.

## `agent/agent.ts` — model + build config

```ts
import { createOpenAICompatible } from '@ai-sdk/openai-compatible';
import { defineAgent } from 'eve';

// Stackbone injects the org's managed OpenRouter sub-key as OPENROUTER_API_KEY.
// eve routes a provider INSTANCE directly; a BARE model string would route
// through the Vercel AI Gateway, which Stackbone does not use.
const openrouter = createOpenAICompatible({
  name: 'openrouter',
  baseURL: process.env.OPENROUTER_BASE_URL ?? 'https://openrouter.ai/api/v1',
  apiKey: process.env.OPENROUTER_API_KEY,
});

export default defineAgent({
  model: openrouter('anthropic/claude-haiku-4.5'),
  // A custom provider carries no context-window metadata, so declare it.
  modelContextWindowTokens: 200_000,
  // Keep @stackbone/sdk + eve external: one SDK invocation context (per-run logs
  // keep their run id) and a complete eve trace (no ERR_MODULE_NOT_FOUND at boot).
  // `stackbone publish` enforces BOTH — an agent that omits them aborts the build.
  build: { externalDependencies: ['@stackbone/sdk', 'eve*'] },
});
```

> `defineAgent` here is from **`eve`**, not `@stackbone/sdk`.

## `agent/instructions.md` — the system prompt

A plain markdown file eve loads by convention as the system prompt — describe the agent's role, tone, and the tools it may call. No code, no frontmatter.

## A tool — `agent/tools/<tool>.ts`

One tool per file, default-exporting `defineTool` from `eve/tools`. The tool's `execute()` is where you reach the ambient `stackbone` client.

```ts
import { defineTool } from 'eve/tools';
import { stackbone, z } from '@stackbone/sdk';
import { escalations } from '../../schema'; // your Drizzle table

export default defineTool({
  description: 'Escalate this lead to a human sales rep.',
  inputSchema: z.object({
    leadId: z.string().describe('CRM contact id'),
    reason: z.string().describe('Short reason for the hand-off'),
  }),
  async execute({ leadId, reason }) {
    await stackbone.database.insert(escalations).values({ leadId, reason });
    return { leadId, tagged: 'needs-human' };
  },
});
```

- `inputSchema` (Zod) validates the arguments the model passes; `execute` destructures them. A no-argument tool uses `inputSchema: z.object({})`.
- Reach any surface — `stackbone.database`, `stackbone.ai`, `stackbone.storage`, `stackbone.rag`, … — from `execute`. See the per-surface deep dives in the main [SKILL.md](../SKILL.md).

## Rules

- **Keep `@stackbone/sdk` and `eve` external** in `build.externalDependencies` — `stackbone publish` aborts otherwise.
- **No HTTP, no Dockerfile.** The runtime serves the agent's signed `/eve/v1/*` session API; you write neither.
- The system prompt is `instructions.md`, not a string in `agent.ts`.

## eve references

- [Introduction](https://eve.dev/docs/introduction) — the agent model.
- [Tools](https://eve.dev/docs/tools) — `defineTool`, input schemas, tool execution.
- [Sessions, runs & streaming](https://eve.dev/docs/concepts/sessions-runs-and-streaming) — how a turn runs and streams.

See [agent-yaml.md](../agent-yaml.md) for the manifest, and [workflow-agents/authoring.md](../workflow-agents/authoring.md) for calling an agent from a workflow.
