# Interview: building an **agent**

An agent is a deep agent — a model + a system prompt + tools — that holds a conversation, running in-process on LangGraph. You scaffolded it with `stackbone init <name> --with agent` or `stackbone add deep-agent <name>`, which writes **one file** (and pins the runtime deps in the root `package.json`):

```
deep-agents/<name>/
  index.ts               ← the WHOLE agent: default export = defineDeepAgent from '@stackbone/sdk/deep'
                            (model + systemPrompt inline + tools + optional interruptOn)
```

Your job here is to fill in **the system prompt** and **the tools** from the user's answers, then move on to the capability checklist. For the exact code shapes follow the **stackbone** skill → `agents/authoring.md` and the `defineDeepAgent` example in its `SKILL.md`.

## The interview — ask one at a time

1. **Role & purpose.** "In one sentence, what does this agent do, and who talks to it?" → frames everything else; opens the `systemPrompt`.
2. **System prompt / behaviour.** "How should it behave — tone, what it must always do, what it must never do, when it should hand off?" → write the `systemPrompt` string (inline; import it from a sibling `.ts` if it grows long). Keep it concrete; this is the most important part.
3. **Tools — the actions it can take.** Tools are how the agent _does_ things (reads data, calls an API, escalates). For **each** tool, ask:
   - **Name & description** — the description is what the model reads to decide when to call it. Make it a clear, action-first sentence.
   - **Inputs** — what arguments does it take? Each becomes a field on the tool's `schema` (a `z.object`), with a `.describe()` the model can see.
   - **What it does & returns** — the tool body. This is usually where a capability gets used (a DB write, an LLM call, a connector) — note which, you'll wire it in step 4. A third-party operation needs no hand-written body: use `connectorTool({ connector, operation })`.
   - **Should a human approve it before it runs?** — if yes, note it for `interruptOn` (the capability checklist covers the session requirement).
   - Repeat until the user has no more tools. **Zero tools is valid** — a pure-conversation agent is fine.
4. **Model.** "Any preference, or default to a fast, cheap model?" Default: `anthropic/claude-haiku-4.5` — a bare id routes through the managed OpenRouter bridge. Suggest a stronger model only if the role needs heavy reasoning.

## Map answers → the config

| Answer               | Lands in                                                                                                                    |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| role + behaviour     | `systemPrompt:` in `deep-agents/<name>/index.ts`                                                                            |
| model choice         | `model:` in the same config (bare id = managed OpenRouter; a LangChain chat-model instance for full control)                |
| each tool            | a `tool()` from `@langchain/core/tools` (with `name`, `description`, `schema`) — or a `connectorTool(...)` — in `tools: []` |
| approval-gated tools | `interruptOn: { <toolName>: true }`                                                                                         |

> A tool's body is the one place you reach the ambient `stackbone` client. Anything the tool needs to read or write — DB, storage, an LLM call, a connector — is a capability. Don't inline it blindly; surface it in the checklist (step 4) so the user confirms it.

## Documentation

- [deepagents (JS)](https://github.com/langchain-ai/deepagentsjs)

## Then → capabilities

When the prompt and tools are sketched, run **[capabilities.md](capabilities.md)** to decide which surfaces each tool needs (database, storage, AI, RAG, connections, prompts, config, secrets). From a tool you can use every surface **except** the workflow-only ones (`requestApproval`, `stackbone.workflows.start/schedule`) — those belong to workflows. Finish by booting `stackbone dev` and chatting with the agent.
