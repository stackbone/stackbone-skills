# Interview: building an **agent**

An agent is a durable eve agent — a model + a system prompt + tools — that holds a conversation. You scaffolded it with `stackbone init <name> --with agent` or `stackbone add agent <name>`, which writes:

```
agents/<name>/
  agent.yaml              ← per-agent manifest; name: MUST equal the folder basename
  package.json
  agent/
    agent.ts              ← model + build config (default export = defineAgent from 'eve')
    instructions.md       ← the system prompt
    tools/                ← one tool per file (default export = defineTool from 'eve/tools')
```

Your job here is to fill in **the system prompt** and **the tools** from the user's answers, then move on to the capability checklist. For the exact code shapes follow the **stackbone** skill → `agents/authoring.md` and the `agent.ts` / tool examples in its `SKILL.md`.

## The interview — ask one at a time

1. **Role & purpose.** "In one sentence, what does this agent do, and who talks to it?" → frames everything else; goes into `instructions.md`.
2. **System prompt / behaviour.** "How should it behave — tone, what it must always do, what it must never do, when it should hand off?" → write `instructions.md`. Keep it concrete; this is the most important file.
3. **Tools — the actions it can take.** Tools are how the agent _does_ things (reads data, calls an API, escalates). For **each** tool, ask:
   - **Name & description** — the description is what the model reads to decide when to call it. Make it a clear, action-first sentence.
   - **Inputs** — what arguments does it take? Each becomes a field on the tool's `inputSchema` (a `z.object`), with a `.describe()` the model can see.
   - **What it does & returns** — the body of `execute()`. This is usually where a capability gets used (a DB write, an LLM call, a connector) — note which, you'll wire it in step 4.
   - Repeat until the user has no more tools. **Zero tools is valid** — a pure-conversation agent is fine.
4. **Model.** "Any preference, or default to a fast, cheap model?" Default: `anthropic/claude-haiku-4.5` via the OpenRouter-compatible provider (see `agent.ts` in the **stackbone** skill). Suggest a stronger model only if the role needs heavy reasoning.

## Map answers → files

| Answer           | Lands in                                                                                                                                       |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| role + behaviour | `agents/<name>/agent/instructions.md`                                                                                                          |
| model choice     | `model:` in `agents/<name>/agent/agent.ts` (keep `build.externalDependencies: ['@stackbone/sdk', 'eve*']`)                                     |
| each tool        | one file `agents/<name>/agent/tools/<tool>.ts`, default-exporting `defineTool` with `description`, `inputSchema` (`z.object`), and `execute()` |

> A tool's `execute()` is the one place you reach the ambient `stackbone` client. Anything the tool needs to read or write — DB, storage, an LLM call, a connector, a sibling agent — is a capability. Don't inline it blindly; surface it in the checklist (step 4) so the user confirms it.

## Documentation

- [eve](https://eve.dev/docs)

## Then → capabilities

When the prompt and tools are sketched, run **[capabilities.md](capabilities.md)** to decide which surfaces each tool needs (database, storage, AI, RAG, connections, prompts, config, secrets, calling another agent). From a tool you can use every surface **except** the workflow-only ones (`requestApproval`, `stackbone.workflows.start/schedule`) — those belong to workflows. Finish by booting `stackbone dev` and chatting with the agent.
