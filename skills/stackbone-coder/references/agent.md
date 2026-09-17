# Interview: building an **agent**

An agent is a deep agent — a model + an instruction + tools — that holds a conversation, running in-process on LangGraph. The instruction is not part of the file: it is a prompt in the box's catalogue, owned by the agent and keyed by its name, written and published in Studio. You scaffolded it with `stackbone init <name> --with agent` or `stackbone add deep-agent <name>`, which writes **one file** (and pins the runtime deps in the root `package.json`):

```
deep-agents/<name>/
  index.ts               ← the WHOLE agent: default export = defineDeepAgent from '@stackbone/sdk/deep'
                            (name + model + tools + optional interruptOn; no instruction — that
                            is a catalogue prompt named after the agent)
```

Your job here is to draft **the instruction text** and build **the tools** from the user's answers, then move on to the capability checklist. The behaviour you gather never becomes a string literal — it becomes the text written into that prompt, in Studio or with `stackbone prompts create`. An agent whose prompt nobody wrote still boots and still answers; Studio marks it as waiting for its text. For the code shape follow the **stackbone** skill; for the full `defineDeepAgent` config read the `Overview` and `Getting started` pages under SDK › Agents (`list_docs`, then `get_doc`).

## The interview — ask one at a time

1. **Role & purpose.** "In one sentence, what does this agent do, and who talks to it?" → frames everything else; opens the instruction.
2. **Instruction / behaviour.** "How should it behave — tone, what it must always do, what it must never do, when it should hand off?" → this is the prompt's text, written to the catalogue, never a string in `index.ts`. Keep it concrete; this is the most important part.
3. **Tools — the actions it can take.** Tools are how the agent _does_ things (reads data, calls an API, escalates). For **each** tool, ask:
   - **Name & description** — the description is what the model reads to decide when to call it. Make it a clear, action-first sentence.
   - **Inputs** — what arguments does it take? Each becomes a field on the tool's `schema` (a `z.object`), with a `.describe()` the model can see.
   - **What it does & returns** — the tool body. This is usually where a capability gets used (a DB write, an LLM call, a connector) — note which, you'll wire it in step 4. A third-party operation needs no hand-written body: use `connectorTool({ connector, operation })`.
   - **Should a human approve it before it runs?** — if yes, note it for `interruptOn` (the capability checklist covers the session requirement).
   - Repeat until the user has no more tools. **Zero tools is valid** — a pure-conversation agent is fine.
4. **Model.** "Any preference, or default to a fast, cheap model?" Default to the scaffold's `openai/gpt-4o-mini`: a bare id the deployment resolves through its model provider. Suggest a stronger model only if the role needs heavy reasoning (`search_docs` "choose a model", area `faqs`).

## Map answers → the config

| Answer               | Lands in                                                                                                                    |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| role + behaviour     | the agent's **prompt in the catalogue**, keyed by its name — never a string in `index.ts`                                   |
| the agent's name     | `name:` in `deep-agents/<name>/index.ts`, equal to the folder name — it is who owns the prompts                             |
| model choice         | `model:` in the same config (a bare id resolved by the model provider; a LangChain chat-model instance for full control)    |
| each tool            | a `tool()` from `@langchain/core/tools` (with `name`, `description`, `schema`) — or a `connectorTool(...)` — in `tools: []` |
| approval-gated tools | `interruptOn: { <toolName>: true }`                                                                                         |

> A tool's body is the one place you reach the ambient `stackbone` client. Anything the tool needs to read or write — DB, storage, an LLM call, a connector — is a capability. Don't inline it blindly; surface it in the checklist (step 4) so the user confirms it.

## Documentation

- `Overview` and `Getting started` under SDK › Agents; a worked example under Examples › Agents
- [deepagents (JS)](https://github.com/langchain-ai/deepagentsjs)

## Then → capabilities

When the prompt and tools are sketched, run **[capabilities.md](capabilities.md)** to decide which surfaces each tool needs (database, storage, AI, RAG, connections, prompts, config, secrets). From a tool you can use every surface **except** the workflow-only ones (`requestApproval`, `stackbone.workflows.start/schedule`) — those belong to workflows. Finish by booting `stackbone dev` and chatting with the agent.
