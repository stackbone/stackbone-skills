# `browserTools()` / `browserSubagent()` — SDK integration

`import { browserTools, browserSubagent } from '@stackbone/sdk/deep'`. These give a **deep agent** a real Chromium page it can navigate, act on, read, and screenshot. `browserTools()` returns tool markers you spread into `defineDeepAgent({ tools: [...] })`; `browserSubagent()` returns a subagent marker you drop into `subagents: [...]`. They mirror `connectorTool` — the SDK returns lightweight markers and `defineDeepAgent` builds the real LangChain tools lazily at graph-build time, so the SDK never statically imports Stagehand or Playwright. Under the hood each tool drives a real page through [Stagehand](https://www.stagehand.dev/); the page-reading tools reason about the DOM through the runtime-injected `OPENROUTER_API_KEY`.

> **Browser tools run under the runtime that binds the browser pool.** Today that is the `stackbone dev` emulator, which binds it automatically. Treat browsing as a **dev-loop** capability: develop and run browser agents with `stackbone dev`. In a host that has **not** bound a pool, every browser call returns `{ ok: false, error: 'No browser session pool is bound…' }` (it never crashes the run).

## Install the peers

The browser tools need two extra workspace deps on top of the deep-agent peers, plus a Chromium binary:

```sh
npm install @browserbasehq/stagehand@3.6.0 playwright@1.59.1
npx playwright install chromium
```

If a `browserTools()` / `browserSubagent()` marker is present but the deps are missing, the agent fails **at graph-build time** with one error naming exactly what to install — not a `module not found` deep inside the first tool call.

## The five tools

`browserTools()` materializes five tools with **fixed, model-facing names** (creators target them in `interruptOn` and in `config.schema.ts`, so they never change):

| Tool                 | Args                       | Uses a model? | What it does                                                    |
| -------------------- | -------------------------- | ------------- | --------------------------------------------------------------- |
| `browser_goto`       | `{ url: string }`          | no            | Navigate to an absolute URL and load the page.                  |
| `browser_observe`    | `{ instruction?: string }` | yes           | List the actions on the current page (buttons, links…).         |
| `browser_act`        | `{ instruction: string }`  | yes           | Do a natural-language action ("click the login button").        |
| `browser_extract`    | `{ instruction?: string }` | yes           | Pull typed, structured data off the page (see `extractSchema`). |
| `browser_screenshot` | `{ fullPage?: boolean }`   | no            | Capture the page as a base64 PNG.                               |

Every tool returns a **JSON string**, `{ "ok": true, … }` on success or `{ "ok": false, "error": "…" }` on failure. A tool **never throws** in a way that crashes the run — a pool-busy/launch failure, a blocked domain, or any Stagehand error comes back as the `{ ok: false }` result the model reads. The model-backed tools (`browser_act` / `browser_extract` / `browser_observe`) default to `anthropic/claude-haiku-4.5` via OpenRouter; `browser_goto` / `browser_screenshot` make no model call.

## Attach it two ways

### `browserSubagent()` — the default

Attaches a ready-made `browser` subagent. The main agent delegates the whole navigation loop (through deepagents' `task` mechanism) and only the final result comes back. Two reasons this is the default:

- **Context isolation** — the click-by-click back-and-forth stays in the subagent's own context instead of flooding the main agent's transcript.
- **Prompt-injection containment by construction** — the subagent carries **only** the five browser tools, so nothing it reads off a page can reach the main agent's connectors, files, or database.

It shares the parent run's browser, so delegating does **not** launch a second Chromium.

```ts
import { defineDeepAgent, browserSubagent } from '@stackbone/sdk/deep';

export default defineDeepAgent({
  model: 'anthropic/claude-sonnet-4.5',
  systemPrompt: 'You research topics. Delegate any web browsing to the browser subagent.',
  subagents: [
    browserSubagent({
      model: 'anthropic/claude-haiku-4.5', // run the navigation loop on a cheaper model
      // allowedDomains: ['example.com'],   // fence in where it may browse
    }),
  ],
});
```

### `tools: [...browserTools()]` — direct tools

Put the five tools on the main agent when browsing **is** the agent's job (no context to isolate), or when you need **per-action human-in-the-loop** on an individual browser action:

```ts
import { defineDeepAgent, browserTools } from '@stackbone/sdk/deep';

export default defineDeepAgent({
  model: 'anthropic/claude-sonnet-4.5',
  systemPrompt: 'You are a web navigator. Browse to complete the task.',
  tools: [...browserTools({ allowedDomains: ['example.com'] })],
  interruptOn: { browser_act: true }, // pause for approval before each action
});
```

`interruptOn` works the same **inside the subagent** — a gated tool in the `browser` subagent still pauses the whole run and surfaces in the approvals inbox (deepagents runs the subagent on the parent's checkpointer). So the subagent-vs-direct choice is about isolation and containment, not about whether approvals work. See [../hitl/sdk-integration.md](../hitl/sdk-integration.md).

## Options

### `browserTools(options?)`

| Option           | Type                | Default                      | Notes                                                                                                             |
| ---------------- | ------------------- | ---------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `model`          | `string`            | `anthropic/claude-haiku-4.5` | Model for `browser_act` / `browser_extract` / `browser_observe`, via OpenRouter. `goto` / `screenshot` ignore it. |
| `extractSchema`  | Zod or JSON Schema  | —                            | Output shape for `browser_extract`, forwarded verbatim to Stagehand so it returns typed data, not raw HTML.       |
| `allowedDomains` | `readonly string[]` | — (allow all)                | Hosts the agent may browse. Fails closed when set (see below).                                                    |

```ts
import { z } from '@stackbone/sdk';
import { browserTools } from '@stackbone/sdk/deep';

const tools = browserTools({
  extractSchema: z.object({ title: z.string(), price: z.number() }), // browser_extract now returns this shape
  allowedDomains: ['example.com'],
});
```

### `browserSubagent(options?)`

| Option           | Type                          | Default                 | Notes                                                                                                             |
| ---------------- | ----------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `model`          | `string` or a LangChain model | inherits the main agent | The subagent's own reasoning loop. A bare id routes through OpenRouter; an instance passes through verbatim.      |
| `allowedDomains` | `readonly string[]`           | — (allow all)           | Threaded into the bundled `browserTools({ allowedDomains })`, so it fences hosts the same way as the direct form. |

## Domain allowlist

By default an agent may browse **any** `http(s)` host — the right default for a local dev tool. Set `allowedDomains` and the tools **fail closed**:

- `browser_goto` refuses a host that is not an exact match or a subdomain of a listed host, and **re-checks the URL it actually landed on** after any redirect (so a prompt-injected redirect can't smuggle the agent off the list).
- `browser_act` / `browser_extract` / `browser_observe` / `browser_screenshot` refuse to act on, read, or capture a page whose current host is off-list.
- Non-`http(s)` schemes (`javascript:`, `data:`, `file:`…) are **always** refused, regardless of the list.

Matching is exact host or dot-boundary subdomain: `example.com` allows `example.com` and `app.example.com`, but not `notexample.com` or `example.com.evil.com`.

**Per-install override.** Declare an `allowedDomains` key in the workspace `config.schema.ts` and an operator can tighten the list from Studio **without a code change**: a non-empty config value wins over the `allowedDomains` you passed in code. An empty or unset config value falls back to the code option — it never silently weakens it.

**Pre-validate a URL** with the same rule via the exported pure predicate (no I/O, no peers):

```ts
import { checkDomainAllowed } from '@stackbone/sdk/deep';

checkDomainAllowed('https://app.example.com/login', ['example.com']); // { allowed: true }
checkDomainAllowed('javascript:alert(1)', ['example.com']); // { allowed: false, reason }
```

## One browser per session

The runtime keeps a **per-session browser pool**: the first browser tool call in a run launches Chromium and caches it under the run's scope; every later call in that scope reuses the same page (cookies, tabs, current page intact). This is what keeps a login alive across a conversation's turns and across a human-in-the-loop pause.

- A **chat conversation** keys the browser to its durable session, so it survives across turns and a pause/resume.
- A **workflow-triggered run** gets a **unique** scope, so two concurrent runs never share one Chromium's cookies.
- A **subagent** inherits the parent run's scope, so delegating reuses the parent's browser.

## Watch it run / remote browser

Two env vars let an operator change the local launch without a code change (the runtime reads them at launch):

| Env var                      | Effect                                                                                    |
| ---------------------------- | ----------------------------------------------------------------------------------------- |
| `STACKBONE_BROWSER_HEADED=1` | Launch a **visible** window, so you can watch a `stackbone dev` browsing agent work.      |
| `STACKBONE_BROWSER_CDP_URL`  | Point the same tools at an **already-running** browser over the Chrome DevTools Protocol. |

## Best practices

1. **Prefer the subagent.** Reach for `browserSubagent()` by default; use direct `browserTools()` only when browsing is the agent's whole job or you need per-action `interruptOn`.
2. **Fence production with `allowedDomains`.** Leave it open for local dev; set it (or an operator's `config.schema.ts` override) the moment the agent runs untrusted content.
3. **Give `browser_extract` a schema.** Pass `extractSchema` so the model gets typed data instead of raw HTML it has to re-parse.
4. **Read the `{ ok }` envelope.** Browser tools return a JSON string, never throw to the graph — the model already sees `{ ok: false, error }` and can recover.
5. **Develop under `stackbone dev`.** That is the runtime that binds the browser pool today.

## Common mistakes

| Mistake                                                              | Fix                                                                                                      |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Adding a browser tool without installing the peers                   | `npm install @browserbasehq/stagehand@3.6.0 playwright@1.59.1` + `npx playwright install chromium`.      |
| Passing `browserTools()` (a marker array) where one tool is expected | Spread it: `tools: [...browserTools()]`. Each entry becomes a real tool at graph-build time.             |
| Expecting browsing to work in a published/cloud agent today          | The pool is bound by the `stackbone dev` emulator; elsewhere calls return `{ ok:false }`. Run it in dev. |
| Assuming an empty `allowedDomains` config **widens** the fence       | An empty/unset config value falls back to the code option; a non-empty one wins. It never weakens it.    |
| Expecting a raw string / HTML back from a tool                       | Every tool returns a JSON string `{ ok, … }`; `browser_screenshot` returns `{ ok, format, base64 }`.     |

See [../connections/sdk-integration.md](../connections/sdk-integration.md) for `connectorTool` (the non-browser tool of the same shape), and the wiki page `/docs/agents/browser-tools` for the reader-facing walkthrough.
