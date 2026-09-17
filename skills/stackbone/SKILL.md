---
name: stackbone
description: >-
  Use this skill when writing the code inside a Stackbone workspace with @stackbone/sdk:
  authoring a deep agent (deep-agents/<name>/index.ts default-exporting defineDeepAgent({ name,
  model, tools, subagents, interruptOn }) from @stackbone/sdk/deep, whose instruction is NOT in the
  code but a catalogue prompt owned by the agent, with LangChain tools and connectorTool(...) for
  third-party operations), writing durable workflows as 'use workflow' /
  'use step' functions in workflows/<name>.workflow.ts with sibling inputSchema / outputSchema,
  calling a sibling agent from a workflow step with callDeepAgent / streamDeepAgent from
  @stackbone/sdk/workflow, and reaching the ambient `stackbone` client from a tool or a step:
  stackbone.database (Drizzle over the agent's Postgres), stackbone.storage, stackbone.ai,
  stackbone.rag, stackbone.config / secrets / settings / prompts, stackbone.workflows (start or
  schedule another workflow) and stackbone.connection(id) for a connector, plus human-in-the-loop
  both ways (requestApproval() in a workflow body, tool-level interruptOn in an agent).
  Trigger on requests like: build an agent, add a tool, write a workflow, store data, upload a file,
  call an LLM, ingest docs for RAG, pause until a human approves, gate a tool behind approval,
  call another agent, call a connector, read dynamic config, give an agent a browser, run a
  workflow serially. For CLI tasks (init, add, dev, build, db migrate, runs, hitl) use the
  stackbone-cli skill; to triage an error or a stuck run use stackbone-debug; to start a new
  piece from a description use stackbone-coder.
license: MIT
metadata:
  author: stackbone
  version: '2.2.0'
  organization: Stackbone
  date: September 2026
---

# Stackbone SDK skill

This skill tells you **how to work** inside a Stackbone workspace. It holds no API reference: every signature, option, error code and limit lives in the Stackbone docs, and you read them from there.

## Where the facts live

The docs are served over MCP as the `stackbone-docs` server (`https://docs.stackbone.ai/mcp`, public, tools `search_docs`, `get_doc`, `list_docs`). The CLI wires it into your coding agent on `stackbone init` / `stackbone link`; `stackbone docs` prints the connection details.

- **Start the session with `list_docs`.** It returns the table of contents: every page under its area and section, with a one-line description. [How to find the page](#how-to-find-the-page) below says which page a task needs; take its path from that list and `get_doc` it. Never type a path from memory: pages move, titles stay.
- Before you write a call to a surface you have not read about in this session, read its page.
- When you do not know which page holds a method, an option, an error code or a limit, `search_docs` with the question in plain words (pass `area` when you know it: `sdk`, `cli`, `examples`, `faqs`, `home`), then `get_doc` the best hit.
- Do not write a signature from memory. A wrong one compiles and fails at runtime in a `'use step'`.

**No `stackbone-docs` tools in your session?** Fetch `https://docs.stackbone.ai/llms.txt`: the same index, one line per page with its title, its description and a link to its raw markdown. Pick the page by title and fetch that link. `https://docs.stackbone.ai/llms-full.txt` is every page in one file (~700 KB).

## The mental model

- A **workspace** is discovered by convention from the files on disk: every `deep-agents/<name>/index.ts` is an agent (the folder name is its name and the `model` a client selects), every `workflows/<name>.workflow.ts` is a workflow (name = file basename, export = `<camelCase(name)>Workflow`). There is no registry to edit.
- A **deep agent** is one file: `export default defineDeepAgent({ name, model, tools })` from `@stackbone/sdk/deep`. Its instruction is not in that file — it is a prompt in the box's catalogue, owned by the agent and keyed by its name, written and published in Studio. It runs in-process and the runtime serves it over the standard OpenAI / Anthropic chat wire. You write no HTTP code.
- A **workflow** is a plain async function marked `'use workflow'` whose side effects live in helper functions marked `'use step'`. Each step is a durable checkpoint: it runs once, is persisted, and is retried on failure.
- Every surface is reached through the **ambient client**: `import { stackbone } from '@stackbone/sdk'`, from any tool body or any step. No `createClient()`, no connection strings.

## How to work

1. **Locate the piece.** The agent is `deep-agents/<name>/index.ts`; the workflow is `workflows/<name>.workflow.ts`. If it does not exist, scaffold it with `stackbone add agent|workflow|workflow-agent <name>` (the **stackbone-cli** skill). Do not hand-write the file layout or the `package.json` pins.
2. **Read before you write.** For each surface the change touches, read its page ([how to find it](#how-to-find-the-page)). Read the whole page: the option you need is usually in the second half.
3. **Write the code** in the shapes below. Keep a tool body or a step small; put every ambient call inside it.
4. **Verify in the emulator.** `stackbone dev`, then chat with the agent or `stackbone workflows start <name> --input '…'`, then `stackbone runs get <runId>` and `stackbone logs tail --run <runId>` (the **stackbone-cli** skill). A change is not done until a run has passed through it.
5. **Something fails?** Switch to the **stackbone-debug** skill to locate the cause, then come back here for the fix.

## The three shapes

### A deep agent: `deep-agents/<name>/index.ts`

```ts
import { tool } from '@langchain/core/tools';
import { z } from 'zod';
import { defineDeepAgent } from '@stackbone/sdk/deep';
import { stackbone } from '@stackbone/sdk';

const readTone = tool(
  async () => {
    const tone = await stackbone.config.get('tone'); // any ambient surface works here
    return tone.error ? 'neutral' : String(tone.data);
  },
  {
    name: 'read_tone',
    description: "Return the agent's current tone setting.",
    schema: z.object({}),
  },
);

export default defineDeepAgent({
  name: 'support', // must equal the folder name — it is who owns this agent's prompts
  model: 'openai/gpt-4o-mini', // a bare id, resolved through the deployment's model provider
  // No instruction here on purpose: it is a prompt keyed by this agent's name, written in
  // Studio. Add `instructions: { key, variables }` to point at another key or feed it {{vars}}.
  tools: [readTone],
  // interruptOn: { send_mail: true }, // pause for a human before this tool runs
});
```

### A durable workflow: `workflows/<name>.workflow.ts`

```ts
import { z } from 'zod';
import { stackbone } from '@stackbone/sdk';
import { welcomes } from '../src/schema'; // your Drizzle tables live at src/schema.ts

export const inputSchema = z.object({ userId: z.string(), email: z.string().email() });
export const outputSchema = z.object({ userId: z.string(), welcomed: z.boolean() });
type OnboardingInput = z.infer<typeof inputSchema>;

export async function onboardingWorkflow(input: OnboardingInput) {
  'use workflow'; // cheap, deterministic glue: it replays on resume
  const valid = await validateSignup(input);
  return await persistWelcome(valid.userId);
}

async function validateSignup(input: OnboardingInput) {
  'use step';
  if (!input.email.includes('@')) throw new Error(`Invalid email: ${input.email}`);
  return { userId: input.userId };
}

async function persistWelcome(userId: string) {
  'use step'; // runs once, persisted, retried: make it idempotent on userId
  await stackbone.database.insert(welcomes).values({ userId }).onConflictDoNothing();
  return { userId, welcomed: true };
}
```

### A workflow that calls an agent: one step delegates the turn

```ts
import { callDeepAgent } from '@stackbone/sdk/workflow';

async function askAgent(message: string) {
  'use step'; // the whole agent turn is this one durable step
  const { text } = await callDeepAgent('support', message);
  return text;
}
```

`streamDeepAgent` (same import) is the streaming twin the `workflow-agent` scaffold uses: the reply streams onto the run's chat surface and Studio shows the workflow as a chat.

## Rules that hold everywhere

- **Destructure `{ data, error }` and handle both branches.** Every ambient surface returns that envelope. `throw` to fail the step, or branch on `error.code` for a case you handle. Never swallow `error`.
- **Three surfaces throw instead.** `stackbone.database` is native Drizzle (rows back, throws on error); `callDeepAgent` / `streamDeepAgent` throw and fail the enclosing step; `stackbone.connection(id)` throws a `ConnectorCallError`, matched by `err.code`, never by `instanceof`.
- **Steps are idempotent.** A `'use step'` may run twice. Write it so that running twice is safe.
- **A workflow that writes through the connector whose trigger started it needs `isOwnEcho`.** That write causes the change the trigger delivers, so the run starts itself again, and deduplication cannot stop it because nothing is being duplicated. Name an `echoPath` on the write, then guard the step before it writes: `if (await isOwnEcho(connectorId, value)) return { … }` (from `@stackbone/sdk/workflow`, and the early return still owes your declared `outputSchema` a value). MATCH `value` TO THE TRIGGER. A trigger that watches for new objects: the new object’s id. A trigger that follows a change column such as `updatedAt`: that column, because an id never changes and a guard on it would swallow every later human edit of that object. Never the id of the event, which is new on every lap. Read the **Trigger events** page under Home before you pick, and take the exact path from the trigger’s own caution.
- **`requestApproval()` runs in the workflow body, never inside a `'use step'`,** and is imported from `@stackbone/sdk/workflow`, not the main barrel. Gate the side effect on `decision.status === 'approved'`.
- **The instruction is never in the code.** It is a prompt in the box's catalogue, owned by the agent and keyed by its name, written and published in Studio (or with `stackbone prompts create`). `instructions: { key, variables }` points at another key or feeds it `{{vars}}` — it never carries the text. An agent whose prompt nobody wrote still boots and answers.
- **The model is a bare id** (`'openai/gpt-4o-mini'`) the deployment resolves through its model provider, or a built LangChain chat-model instance. Do not construct a client with a key.
- **Runtime dependencies live in the workspace root `package.json`** (`deepagents`, `@langchain/*`, `workflow`), one copy per process. No per-agent `package.json`, no nested `node_modules`.
- **Never hardcode or ask for injected env.** `STACKBONE_POSTGRES_URL`, `MODEL_PROVIDER_API_KEY`, `MODEL_PROVIDER_BASE_URL`, `STACKBONE_INSTALLATION_ID`, storage credentials: the runtime injects them. Operator-managed values go through `stackbone.secrets` and `stackbone.config`.
- **Declare the workflow contract as sibling `inputSchema` / `outputSchema` exports.** The input type derives from `inputSchema` with `z.infer`; the output is declared, not inferred.
- **The RAG schema is platform-provisioned.** Your own tables come from `src/schema.ts` + `stackbone db migrate create` (the **stackbone-cli** skill); never hand-write the RAG tables.

## How to find the page

The docs follow the code's names, so you rarely need to search:

- **An ambient surface has a page titled like it** under SDK: `stackbone.database`, `stackbone.storage`, `stackbone.rag`, `stackbone.ai`, `stackbone.config`, `stackbone.secrets`, `stackbone.settings`, `stackbone.prompts`, `stackbone.connection`. Before you write a call to one, `get_doc` that page (path from `list_docs`). Two break the rule: `stackbone.workflows` is documented on `Background jobs & workflow triggers` and `requestApproval` / the approval surface on `Human-in-the-loop`.
- **Each shape has an `Overview` and a `Getting started`** under its SDK section (Agents, Workflows, Workflow + Agents). Read both the first time you author that shape in a session; the section's other pages (`Calling a sibling agent`, `Browser tools`, `Serial execution`) sit next to them in the list.
- **A helper or a type with no page of its own** (`interruptOn`, `callDeepAgent`, `callConnector`, `connectorTool`, `defineWorkspace`, `Result<T>`, an error code): `search_docs` the identifier verbatim with area `sdk` and take the top hit.
- **A worked example**: `list_docs` with area `examples`. **A product question** (agent or workflow? which model?): area `faqs`.

## Other skills

- **stackbone-cli**: scaffold (`init`, `add`), run (`dev`), migrate (`db migrate`), operate runs / approvals / logs, ship (`build`, `package`, `link`).
- **stackbone-debug**: locate the cause of an error, a failed or stuck run, a parked approval.
- **stackbone-coder**: start a new piece from a one-line idea through an interview, then come back here for the code.
