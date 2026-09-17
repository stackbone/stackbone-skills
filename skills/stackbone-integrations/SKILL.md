---
name: stackbone-integrations
description: >-
  Use this skill when a Stackbone agent or workflow has to reach the outside world or be reached
  by it: registering a connector and connecting the account behind it, calling a provider
  operation from a tool or a step, arming a trigger so a provider event starts a durable workflow
  run, mapping an event's fields onto a workflow's input, wiring a correlation key so a reply
  resumes the run that was waiting for it, scoping a trigger to a folder or a mailbox, declaring
  what a workflow emits and binding each output to a destination, and reading why an output is
  parked instead of filed. Trigger on requests like: make my agent read my email, connect Gmail,
  connect Drive, connect a custom OpenAPI service, start a workflow when a file lands, my trigger
  is not firing, the trigger fired but no run started, the reply started a new run instead of
  resuming, where does the report go, send the output to a Drive folder, bind an output, my output
  is parked, the connector is missing a permission, I renamed an output and nothing files. This
  skill drives the wiring; for the code inside the workspace use the stackbone skill, for the rest
  of the CLI use stackbone-cli, and to triage a run that failed use stackbone-debug.
license: MIT
metadata:
  author: stackbone
  version: '2.2.0'
  organization: Stackbone
  date: September 2026
---

# Stackbone integrations skill

This skill tells you **how to wire** a workspace to the outside: what to set up first, which surface can set it up, and what each step commits you to. It holds no connector catalog and no field lists: which providers exist, which events they fire and which errors they answer with live in the docs and on the box.

## Where the facts live

The docs are served over MCP as the `stackbone-docs` server (`https://docs.stackbone.ai/mcp`, public, tools `search_docs`, `get_doc`, `list_docs`). `stackbone docs` prints the connection details.

- **Start with `list_docs`.** `Integrations`, `Trigger events` and `Destinations` (Home › Features) are the three pages behind this skill. Take their paths from that list and `get_doc` them. Never type a path from memory: pages move, titles stay.
- **The connector catalog is not in the docs, it is on the box.** Which connectors are registered here, which accounts are connected and which operations each one exposes is read live from the provider's own OpenAPI document. A list read from a doc is a list from another release.
- **Every destination fact is on the `Destinations` page**: the kinds, what a binding is keyed by, what a redeploy does to it, the refusal codes.

**No `stackbone-docs` tools in your session?** Fetch `https://docs.stackbone.ai/llms.txt`: the same index, one line per page with its title, its description and a link to its raw markdown. Pick the page by title and fetch that link.

## Reach the box

Connections, trigger links and bindings are box state, so point your client at the box once:

```sh
claude mcp add --transport http my-box http://127.0.0.1:4242/mcp   # a local box under `stackbone dev`
```

A deployed box takes its own address and nothing else: the client negotiates the sign-in itself. `MCP` (Home › Features) has the deployed and self-hosted variants.

Then work by name, never by sweep. `list_operations` is capped and answers `catalog_too_large` past it, so always pass a tag — `connect` for connections, `trigger-subscriptions` for trigger links, `destinations` for output bindings — read the ids it returns, and `describe_operation` the one you want. Reads go through `call_read_operation` and writes through `call_write_operation`; the wrong one is refused with `wrong_operation_kind` before anything runs.

## What each surface can do

The single most useful fact in this skill: **the CLI cannot wire any of this.** It has no connectors command. It starts a workflow with the input an event would have carried, and that is its whole part in the loop.

| Step                                          | Code    | CLI     | Box MCP | Studio                |
| --------------------------------------------- | ------- | ------- | ------- | --------------------- |
| Register a connector, add its credential      | no      | no      | **yes** | **yes**               |
| Hand you the provider's consent link          | no      | no      | **yes** | **yes**               |
| Finish the provider's sign-in                 | no      | no      | no      | a human, in a browser |
| Call a provider operation                     | **yes** | no      | yes     | no                    |
| Arm a trigger onto a workflow                 | no      | no      | **yes** | **yes**               |
| Draw or dry-run a mapping                     | no      | no      | yes     | **yes**               |
| Declare what a workflow emits                 | **yes** | no      | no      | no                    |
| Bind an output to a destination               | no      | no      | **yes** | **yes**               |
| Drain a parked output                         | no      | no      | yes     | **yes**               |
| Start the workflow by hand, no account needed | no      | **yes** | yes     | yes                   |

## Zero to an event that lands

Inbound, in this order. Each step is unusable until the one above it exists.

1. **Register the connector** and give it its credential. `owner` or `admin`: a connection holds a customer credential, so it follows the same gate as secrets.
2. **Connect the account**, if the provider authenticates with OAuth. A key-based provider is ready on registration; an OAuth one says **No account** until the sign-in is finished. Ask the box for the consent link and a person opens it in their own browser: the box only learns the account once they accept, and the link is single-use and expires in minutes. That grant is who the box acts as.
3. **Create the trigger link**: the connector's event, the workflow it starts, which of the connection's credentials to poll with (the shared account unless you say otherwise), and any value the trigger declares to scope the watch (a Drive folder id, an App-Only mailbox). Studio's dialog creates it switched **off**; over the box you can send the mapping and the switch in the same call, which is the safer shape — see the first rule below.
4. **Map the event onto the workflow's input** and save it. The event's fields and the workflow's `inputSchema` sit side by side; a transform goes between them when a value needs shaping. If the reply to something this workflow sent should resume the run rather than start a new one, wire the field that identifies the conversation to the **correlation key**, and park the workflow's hook on the same value.
5. **Turn it on, last.** Arming stamps the polling cursor at that moment, so the link starts from now. Saving a mapping afterwards keeps that cursor where it is rather than re-stamping it, which is why the docs say a save does not replay what the box already handled.

Outbound is two halves that never meet in one file: the workflow exports `declaredOutputs` with literal names and a kind, and an operator binds each `(workflow, output name)` to a connector, an account and — for a `file` — a folder. Then a run writes by name, and the box files it.

## The vocabulary

- **Connection.** A registered connector plus the account the broker acts as. The credential is encrypted inside your box, never read back to a screen and never handed to your code: you name the connector and the operation, the broker mints a short-lived token for that one call.
- **Trigger link.** One provider event bound to one workflow, polled on a timer. The link owns the cursor, the mapping and the on/off switch.
- **Mapping.** How an event's fields fill the workflow's declared input. Saved as a **JSONata** expression — `{"orderId": subject}` reads a field of the event by its name — and drawn as a graph. **The two are stored separately, and the canvas is drawn only from the graph.**
- **Delivery.** One event claimed under the provider's own id, so the same item is never delivered twice. A delivery that the workflow rejects is still a delivery.
- **Declared output.** A name the workflow's code exports as something it will write, with `kind: 'file'` or `kind: 'action'`. Names must be literal, because an operator binds to one before any run exists.
- **Binding.** The row an operator creates against that name. Keyed by workflow and output name, one per name, and it lives in the box's database rather than in the bundle.
- **Parked.** Written but not filed, and waiting for a person. Nothing is dropped.

## Rules that bite

- **Save the mapping before you turn the link on.** A link armed with no saved mapping polls happily: the event arrives, the workflow rejects the input, the delivery is marked rejected and the cursor advances anyway. A rejection is deterministic, so it never retries — the item is gone. Only a new item at the provider gets you another run. Creating the link with its mapping and its switch in one call closes that window; dry-run the mapping against the trigger's sample event first, and it answers with the input the workflow would receive.
- **A blank mapping canvas is not an empty mapping.** A mapping written as an expression by API or in Code mode has no graph, so the editor opens empty while the saved expression is still working. One **Save mapping** there compiles the blank canvas to identity and overwrites both the mapping and the correlation key. Read the link over the API before you touch that screen, not the canvas.
- **Arming starts from now.** The cursor is seeded at arm time, from the provider where only it can mint the value, so a mailbox armed today does not replay its history. Drive is the exception worth knowing: it asks on the arrival date as well as the edit date, so a document dropped into the folder starts a run however old the document itself is, and one that was already sitting there arrives the first time anyone touches it.
- **Never file into the folder you watch.** A file the connector writes is an arrival like any other, and each upload is a new id, so deduplication cannot stop it: the run writes, the write fires the trigger, and it does not end. A workflow that creates the file itself can guard with the connector's own echo check; the output writer behind a destination binding cannot carry one, so there the only answer is two folders.
- **A connector's scopes freeze when it is registered.** The authorize URL is built from the registered row, not from the catalog. Adding a scope means registering the connector again and then reconnecting the account; reconnecting alone re-grants exactly the old scopes.
- **Renaming an output orphans its binding.** The old row survives under the old name and the new name has none, so the next run parks. Bind the new name, then remove the old row. A redeploy on its own changes nothing.
- **Draining is one row, pressed by a person.** There is no sweep, and filing is deduplicated per run and output, because creating a document at a provider is not a repeatable act.

## What to tell the user

Say it before the first write, in two or three sentences, and never paste an operation id or a raw payload back at them.

- **What you are about to wire**, in their words: what will arrive, what will start, and where what it writes will land.
- **Which screen shows it**, named the way Studio's sidebar names it — **Connections**, **Triggers**, **Destinations** — so they can watch the link poll and the file get filed.
- **The one thing only they can decide**: which account to authorise, which folder or mailbox to watch, which folder the output lands in. Ask for that and nothing else.

Afterwards, one line: what is now armed, which screen shows it, and how to switch it off.

## What this skill does not cover

API keys, the fourth row of the Integrations group: that is how a caller gets into the box, not how the box reaches out. Writing the agent or the workflow being wired. Which connector to buy your way into — the catalog in Studio is the current answer.

## Other skills

- **stackbone**: the code inside the workspace — `connectorTool`, `stackbone.connection`, `callConnector`, `declaredOutputs`, and parking a run on a correlation key.
- **stackbone-cli**: the rest of the CLI, including `workflows start` to exercise the path before a real account exists.
- **stackbone-debug**: a delivery that arrived and a run that then failed.
