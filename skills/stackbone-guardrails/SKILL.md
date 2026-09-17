---
name: stackbone-guardrails
description: >-
  Use this skill when putting limits on a Stackbone agent or workflow that is already running:
  blocking a turn whose message breaks a rule, masking personal data or credentials out of a
  message before the model reads it, holding a turn for a person to approve, choosing between a
  deterministic check and a model-backed one, picking the checking model the box judges with,
  attaching a rule to one agent, one workflow or the whole workspace, deciding what happens when
  the check itself cannot run (fail open or fail closed), telling a content rule apart from
  human-in-the-loop, reading a tripped rule afterwards on the run's own trace,
  deciding a held turn from the inbox, and narrowing or removing a rule without losing it. Trigger
  on requests like: stop my agent leaking emails, block prompt injection, add a guardrail, mask
  credit card numbers, my agent answered something it should not have, moderate the input, hold
  this for approval before it sends, why is my guardrail not firing, everything is blocked now,
  redact PII, restrict what my agent talks about, review the reply before the user sees it, turn
  off a rule without deleting it. This skill drives the guardrail loop; for the code inside the
  workspace use the stackbone skill, for the rest of the CLI use stackbone-cli, to measure whether
  the agent is any good use stackbone-evals, and to triage a failure use stackbone-debug.
license: MIT
metadata:
  author: stackbone
  version: '2.2.0'
  organization: Stackbone
  date: September 2026
---

# Stackbone guardrails skill

This skill tells you **how to put a limit in place** and how to tell why one is not firing. It holds no check catalog and no field lists: the checks and their settings are read live off the box, and everything else lives in the docs.

## Where the facts live

The docs are served over MCP as the `stackbone-docs` server (`https://docs.stackbone.ai/mcp`, public, tools `search_docs`, `get_doc`, `list_docs`). `stackbone docs` prints the connection details.

- **Start with `list_docs`.** `Guardrails` (Home › Features) is the operator page and `Guardrails` (SDK › Humans) is the developer one. Take their paths from that list and `get_doc` them. Never type a path from memory: pages move, titles stay.
- **The check catalog is not in the docs, it is on the box.** Read it before you write a rule. It is the same schema the Studio form is generated from and the same one the runtime validates against, so a setting the catalog accepts is a setting that will run. A list of checks copied from a page goes stale the day a new one ships.

**No `stackbone-docs` tools in your session?** Fetch `https://docs.stackbone.ai/llms.txt`: the same index, one line per page with its title, its description and a link to its raw markdown. Pick the page by title and fetch that link. `https://docs.stackbone.ai/llms-full.txt` is every page in one file (~700 KB).

## There is nothing to write in code

A guardrail is not a function and there is no `stackbone.guardrails` in the SDK. It is a row in the box's own database, read on the turn that needs it, which is why an edit takes effect on the next turn with no rebuild and no restart.

There is also no CLI command for it. The only guardrail-adjacent thing the CLI does is operate the inbox, with `stackbone hitl`, and that decides a held turn rather than touching the rule. Everything else is Studio or the box's own API.

Point your client at the box once:

```sh
claude mcp add --transport http my-box http://127.0.0.1:4242/mcp   # a local box under `stackbone dev`
```

A deployed box takes its own address and nothing else: the client negotiates the sign-in itself. `MCP` (Home › Features) has the deployed and self-hosted variants. Then work by name, never by sweep: `list_operations` is capped and answers `catalog_too_large` past it, so pass `tag: "guardrails"`, read the ids it returns, and `describe_operation` the one you want. Reads go through `call_read_operation` and writes through `call_write_operation`.

## The five nouns

- **The rule** is the stored row: a name, a check, a side, an action, its settings, an on switch and a fail-open switch. The name is what a refusal quotes back at the user, so write it for a stranger.
- **The check** is the predicate. Some are pure and local; some ask a model. A check answers pass, violation, or "could not run", and it never decides what happens next.
- **The side** is `input` or `output`: the message going in, or the reply coming out. A check can only guard the sides it declares, and the box refuses a save that points it at the other one.
- **The action** is what the box does with a violation. Block ends the turn. Redact rewrites the content and carries on, and the rewritten text is what the model reads **and** what the run records. Hold parks the turn for a person, and is the one whose stored value does not match its name: the box calls it `require_approval`.
- **The attachment** is where the rule applies: one agent, one workflow, or the whole workspace. Agent and workflow are different kinds even when they share a name.

## Zero to a rule that fires

1. **Decide the unit and the side.** An `output` rule buffers the whole reply until it is judged, so that agent loses token-by-token streaming. Keep output rules to the agents that need them.
2. **Read the box's catalog** and pick the check from there, with its settings. Do not type setting names from memory.
3. **If the check asks a model**, make sure the box has a model provider. The checking model itself is optional: left unset it falls back to a platform default, so what bites is the provider — or a checking model you named that the provider will not serve. A check that cannot run fails closed by default, so **every guarded turn blocks**, and the refusal names the check rather than anything about your content.
4. **If the action is hold**, the chat has to carry a durable session. Without a thread there is nothing to replay later, so the box ends the turn instead of parking it.
5. **Create the rule**, on the box that runs the agent — rules are that box's own rows, so a rule written against your laptop changes nothing in production. Studio's form, or a write against the box. It needs a seat that can write config, which owner, admin and member have. Approver and viewer get the screen read-only rather than an error. **A create sent to the box with no attachment lands on the whole workspace**, live from the next turn: Studio's form makes you choose where, the API chooses for you.
6. **Attach it**, if you did not scope it on the way in. This is the step everyone skips, and a rule attached nowhere does nothing — the list says so by showing it applies nowhere.
7. **Watch it fire** in the Playground. A blocked turn ends without an error: the stream closes normally and the reason arrives as the reply.

## Why a rule is not firing

Every one of these is silent. Walk them in order.

- **It is not attached anywhere.** The commonest by a distance.
- **It is switched off.**
- **The kinds do not match**: a rule attached to an agent will not fire on a workflow of the same name.
- **The sides do not match**: it guards the message and you are watching the reply.
- **The call was platform-invoked** (a document ingest, a trigger's mapper, an eval criterion). The exemption follows the call, never the name.
- **The box could not read its own policy.** Then the turn ran with no rules at all, and only a warning in the box log says so. This is a layer above fail-open, and fail-open has no say in it.

## Rules that bite

- **Masking runs before judging, always.** So a moderation check reads the text the model would actually receive, not the raw message. Two masking rules compose instead of fighting.
- **The first violation wins**, in an order that does not depend on whether the operator picked block or hold. "Which rule stopped me" is therefore stable.
- **Fail open covers exactly one case**: the check could not run. A real violation still blocks. Read the failure cause on the decision to tell "your content broke a rule" from "the checking model died".
- **Only the personal-data check can redact.** The box refuses the redact action on anything else at save time.
- **A workflow cannot hold.** There is no thread and nobody waiting, so a hold on a workflow becomes a plain refusal.
- **An edit lands on the next turn on the box you wrote through**, and within a few seconds on any other. Do not wait longer than that before assuming something else is wrong.
- **Deleting and detaching both ask you first, through the MCP.** The box's own API does neither: a raw call goes straight through. They are the two operations that stop something being blocked. Detach narrows where a rule applies; delete takes the rule and all of its places with it.

## Holding is the same inbox, not a fourth system

A held turn writes a row in the same approvals table, and shows up in the same inbox, as a workflow's `requestApproval()` and an agent's gated tool. They are told apart by a marker on the row, and the inbox badges the guardrail ones.

What differs is what resuming means. Approving a held turn **replays the message verbatim and does not judge it again**, because a person already said yes. Rejecting it ends the turn with a reply naming the rule.

## Where you see it afterwards

| Surface            | What it carries                                                                                                                                                                               |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Run trace          | A guardrail step beside the model and tool calls, with the action, the reason and the rule. There is no list of firings anywhere: a blocked turn is a normal completion, so you go run by run |
| Activity log       | Only writes that happened: created, updated, deleted, attached, detached                                                                                                                      |
| HITL inbox         | Held turns, badged, with the held message and the rule's own sentence                                                                                                                         |
| The catalog screen | The other direction: open an agent and see every rule governing it                                                                                                                            |

A redaction records the kinds it masked and never the values, because that row is readable by anyone with trace access.

## What to tell the user

Say it before you create or attach anything, in two or three sentences, and never paste an operation id or a raw payload back at them.

- **What the rule will do**, in their words: what it reads, what it stops, and what the person on the other end sees when it fires.
- **Where they will see it**: **Settings → Guardrails** for the rule itself, the run trace for a turn it stopped, the **Activity** log for the change you just made.
- **The one thing only they can decide**: how wide it applies, and whether a check that cannot run should block or wave things through.

Afterwards, one line: what is in force, where it applies, which screen shows it, and that detaching narrows it while deleting takes it away.

## What this skill does not cover

Input validation, which is your schema's job and not a guardrail's. Rate limits and spend caps. Model choice, beyond picking the one that does the checking.

## Other skills

- **stackbone**: the code inside the workspace, including `requestApproval()` and a gated tool, which are the other two ways a turn waits for a person.
- **stackbone-cli**: the rest of the CLI, including `hitl` for deciding a held turn.
- **stackbone-evals**: measuring the agent rather than limiting it. A case a rule stopped counts as stopped, not failed.
- **stackbone-debug**: a failure that is not the agent breaking a rule.
