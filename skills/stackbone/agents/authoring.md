# Authoring a deep agent

An **agent** is a [deepagents](https://github.com/langchain-ai/deepagentsjs) (LangGraph) agent: a model + system prompt + tools that hold a conversation across turns. It runs **in-process** inside the runtime — no server, no port, no build step of its own. One folder under `deep-agents/<name>/` is one agent; the folder name is the agent's name. Scaffold it with `stackbone add deep-agent <name>` (see the **stackbone-cli** skill); this page is the reference for what the file contains.

## Layout

```
deep-agents/<name>/
  index.ts               ← the WHOLE agent: default export = defineDeepAgent({ ... })
```

There is no per-agent `package.json`, no `instructions.md`, no `tools/` folder and no auth/channel plumbing. The runtime deps (`deepagents`, `@langchain/*`) live in the **workspace root** `package.json` — `stackbone add deep-agent` pins them there so the process resolves exactly one copy.

## `index.ts` — the whole agent

```ts
import { tool } from '@langchain/core/tools';
import { defineDeepAgent, connectorTool } from '@stackbone/sdk/deep';
import { stackbone, z } from '@stackbone/sdk';

const escalate = tool(
  async ({ leadId, reason }) => {
    await stackbone.database.insert(escalations).values({ leadId, reason });
    return JSON.stringify({ leadId, tagged: 'needs-human' });
  },
  {
    name: 'escalate',
    description: 'Escalate this lead to a human sales rep.',
    schema: z.object({
      leadId: z.string().describe('CRM contact id'),
      reason: z.string().describe('Short reason for the hand-off'),
    }),
  },
);

export default defineDeepAgent({
  model: 'anthropic/claude-haiku-4.5',
  systemPrompt: 'You are the support agent. Escalate when a human is needed.',
  tools: [escalate, connectorTool({ connector: 'slack', operation: 'chat.postMessage' })],
});
```

### The config

| Field          | Required | What it is                                                                                                                                                                                                                                                           |
| -------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `model`        | yes      | A bare model id string (`'anthropic/claude-haiku-4.5'`) routes through the **managed OpenRouter bridge** — the org's sub-key is injected at runtime as `OPENROUTER_API_KEY`. Pass a built LangChain chat-model instance instead for another provider / full control. |
| `systemPrompt` | yes      | The system prompt, **inline** (or imported from a sibling `.ts` file for long prompts — never a `.md` loaded by convention).                                                                                                                                         |
| `tools`        | no       | LangChain tools (`tool()` from `@langchain/core/tools`) and/or `connectorTool(...)` markers (see [connections/sdk-integration.md](../connections/sdk-integration.md)).                                                                                               |
| `subagents`    | no       | deepagents sub-agent configs, passed verbatim to `createDeepAgent`.                                                                                                                                                                                                  |
| `interruptOn`  | no       | Gate tools behind human approval: `{ <toolName>: true }` (or a LangChain `InterruptOnConfig` object). See [hitl/sdk-integration.md](../hitl/sdk-integration.md).                                                                                                     |
| `backend`      | no       | The agent's file backend. Defaults to the Stackbone storage backend (files persist through `stackbone.storage`); override only if you know why.                                                                                                                      |

## A tool

Tools are plain **LangChain tools**: the Zod `schema` validates the arguments the model passes, and the body destructures them. Inside the body you reach any ambient surface — `stackbone.database`, `stackbone.ai`, `stackbone.storage`, `stackbone.rag`, … (see the per-surface deep dives in the main [SKILL.md](../SKILL.md)). Return a **string** (LangChain tool outputs are messages — `JSON.stringify` structured results).

## How the agent is served

You write no HTTP code. The runtime builds the LangGraph graph once (cached, hot-swapped on save under `stackbone dev`) and serves every agent in the workspace on **one server** over the standard wire:

- `POST /openai/v1/chat/completions` and `POST /anthropic/v1/messages` — the `model` field in the request body selects the agent by name.
- `GET /openai/v1/models` / `/anthropic/v1/models` — the catalog of agents (for client dropdowns).
- Stateless by default (the client replays history); a client that sends an `x-stackbone-session` header gets a **durable server-side session** — it then sends only the new message. See the **stackbone-cli** skill (`dev` reference) for curl examples.

## Rules

- **Keep the runtime deps at the workspace root.** `deepagents` / `@langchain/*` / `@stackbone/sdk` must resolve to one copy per process; `stackbone publish` bundles each agent with them external and aborts if the SDK gets inlined.
- **The folder name is the agent name.** `deep-agents/support/index.ts` is the agent `support` — there is no name field.
- **Side-effectful tools should be idempotent.** A retried turn re-runs the whole turn (same trade-off as workflow steps).

## References

- [deepagents (JS)](https://github.com/langchain-ai/deepagentsjs) — the agent model (`createDeepAgent`, subagents, middleware).
- [LangChain tools](https://docs.langchain.com/oss/javascript/langchain/tools) — `tool()`, Zod schemas, tool execution.

See [workflow-agents/authoring.md](../workflow-agents/authoring.md) for calling an agent from a workflow, and [hitl/sdk-integration.md](../hitl/sdk-integration.md) for gating tools behind approval.
