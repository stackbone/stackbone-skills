---
name: stackbone-evals
description: >-
  Use this skill when measuring whether a Stackbone agent or workflow actually works, and whether
  a change made it better or worse: turning a Playground exchange into a saved eval case, building
  a case list and versioning it, writing a suite (its target, its criteria, its repeats, its pass
  mark), choosing between a deterministic scorer and an LLM judge, wiring a custom workflow as the
  judge, enabling the persona simulator so a case that asks the user back still gets measured,
  judging behaviour rather than wording with the trajectory criteria (which tools, which workflow
  steps, which subagents), launching a run from the CLI as a CI gate, launching a two-variant
  comparison from the box, reading a run's verdicts (passed / failed / errored / stopped) and its
  per-case reasons, comparing a run against the previous one, and cancelling a run that is spending
  budget you did not mean to spend. Trigger on requests like: how do I test my agent, write me an
  eval, is my agent any good, did my prompt change help, set up an eval suite, add a test case from
  this run, run my evals in CI, why did this case fail, compare two versions of my agent, my eval
  run is stuck, gate the deploy on evals, score the answers, add an LLM judge, my agent asks
  questions so the eval never finishes. This skill drives the eval loop; for the code inside the
  workspace use the stackbone skill, for the rest of the CLI use stackbone-cli, and to triage a
  failure use stackbone-debug.
license: MIT
metadata:
  author: stackbone
  version: '2.2.0'
  organization: Stackbone
  date: September 2026
---

# Stackbone evals skill

This skill tells you **how to drive** the eval loop: what to build first, which surface can build it, and how to read the verdict. It holds no criterion catalog and no flag tables: the criteria, their settings and every exit code live in the docs, and the live catalog lives on the box itself.

## Where the facts live

The docs are served over MCP as the `stackbone-docs` server (`https://docs.stackbone.ai/mcp`, public, tools `search_docs`, `get_doc`, `list_docs`). `stackbone docs` prints the connection details.

- **Start with `list_docs`.** `Evaluation` (Home › Features) is the concept page and `eval` (CLI › Reference) is the command page. Take their paths from that list and `get_doc` them. Never type a path from memory: pages move, titles stay.
- **The criterion catalog is not in the docs, it is on the box.** Ask the box for it before you write a suite: the same catalog carries the judging models and the judge workflows this workspace actually has, painted in live. A catalog read from a doc is a catalog from another release.
- `--help` on the binary beats the docs for the installed version. Do not explain an exit code from memory.

**No `stackbone-docs` tools in your session?** Fetch `https://docs.stackbone.ai/llms.txt`: the same index, one line per page with its title, its description and a link to its raw markdown. Pick the page by title and fetch that link. `https://docs.stackbone.ai/llms-full.txt` is every page in one file (~700 KB).

## Reach the box

Most of this loop is not in the CLI. Point your client at the box once:

```sh
claude mcp add --transport http my-box http://127.0.0.1:4242/mcp   # a local box under `stackbone dev`
```

A deployed box takes its own address and nothing else: the client negotiates the sign-in itself. `MCP` (Home › Features) has the deployed and self-hosted variants.

Then work by name, never by sweep. `list_operations` is capped and answers `catalog_too_large` past it, so always pass `tag: "evals"`, read the ids it returns, and `describe_operation` the one you want. Reads go through `call_read_operation` and writes through `call_write_operation`; the wrong one is refused with `wrong_operation_kind` before anything runs.

## What each surface can do

The single most useful fact in this skill: **the CLI cannot build an eval.** It launches a suite somebody else wrote and turns the result into an exit code.

| Step                                           | CLI     | Box MCP | Studio  |
| ---------------------------------------------- | ------- | ------- | ------- |
| Model provider, judging model, simulator model | no      | no      | **yes** |
| Case lists, cases, CSV import, version history | no      | **yes** | yes     |
| Suites and their criteria                      | no      | **yes** | yes     |
| Dry-run one criterion's settings               | no      | **yes** | no      |
| Launch a plain run and gate on it              | **yes** | yes     | yes     |
| Launch a two-variant comparison                | no      | **yes** | yes     |
| Read the variant-by-variant table              | no      | no      | **yes** |
| Cancel a run                                   | no      | **yes** | yes     |

## Zero to a comparison

1. **Settings first.** A run measures nothing without a model provider. A judging model and a simulator model are optional — unset, each falls back to a platform default — but set them anyway, and set them apart, so the judge's variance and the simulated user's do not pool. Studio only. **Launching also takes `admin` or `owner`**, which is how a domain expert can build the cases without holding the button that spends.
2. **Read the box's catalog.** It tells you which criteria exist here and which models and workflows you may name.
3. **Get cases.** Create a list, then seed it from reality: open a real run in Studio and press **Save as test case**. That button lives on the ordinary run detail screen, not under Evals, and it appends one case with the run's exact input and a link back to its trace. Add a persona to any case where the agent asks the user something.
4. **Write the suite**: its target, its case list (pin the version if the score has to mean something next month), its criteria, its repeats, its per-run cap and its pass mark. Dry-run a criterion first if its settings are fiddly.
5. **Smoke it.** `stackbone eval <suite> --agent <id> --json --git-sha "$(git rev-parse HEAD)"`. This is the baseline and the CI gate.
6. **Decide how the second configuration exists.** The model is built into the bundle, so comparing two models means shipping the agent twice and pointing a column at each. The instruction is different and the difference matters: it lives in the prompt catalogue on the box, but a run always reads the **published** version, so no column can pin an older one. To answer "did my prompt change help", publish version A, run the suite, publish version B, run it again, and read the second run against the first. **Publishing changes what live traffic gets while you measure**, so do it on a box nobody is using, or agree the window with whoever owns the agent.
7. **Launch the comparison from the box**, two variants, distinct labels. Then read the side-by-side table in Studio's run detail.

## The vocabulary

- **Scorer and judge.** A scorer is one criterion: a check that turns a case's answer into pass, fail or "nothing to judge here". Most are deterministic. One of them asks a model with your rubric, and that model is the judge. A case passes only when every criterion passes.
- **Dataset version.** A case list is copy-on-write: every edit mints an immutable version, and a run seals the version it measured. Without that, "we scored 90 this month and 70 last month" could just mean somebody deleted the three hard cases. A rename mints nothing, because a rename did not change what was measured.
- **Variant.** A labelled configuration measured against the same cases. Two at most, labels distinct. The box honours pointing column B at a second target of the suite's own kind — a second agent, or a second workflow — and a different rubric version. It refuses a different model, prompt version or judging model, and it refuses them **before** the run opens, so you never pay for a comparison whose two halves were identical. A pinned prompt version is refused for a reason worth knowing: nothing on the run path can ask for one, so the column would have measured the published version twice.
- **Persona simulator.** Who answers when a case stops to ask. Without it, an agent that clarifies before acting parks on its first question and the half you cared about was never measured. The case's own replies are replayed first, verbatim and in order; the model only improvises once they run out.
- **Trajectory.** What the run did, as three lists of names: tools called, workflow steps run, subagents delegated to. It is for the criteria that judge behaviour rather than wording. Arguments are never compared, only names and order.

## Rules that bite

- **Only the box MCP asks you first.** There it comes back as a confirmation because a launch spends the workspace budget, and a client that cannot be asked gets `confirmation_unavailable` and nothing runs. Studio shows the plan and the rough spend before the button. **The CLI does neither**: `stackbone eval` is not one of the verbs behind `--yes`, so it starts spending the moment you press return. Read the plan yourself first — cases × repeats × variants, doubled again by a judge.
- **Cancelling is deliberately the cheaper gate.** A brake must never be harder to reach than the pedal.
- **`Ctrl-C` and `--timeout` stop the CLI's watcher, not the run.** The box keeps measuring and keeps spending. Cancel it properly.
- **The plan is cases × repeats × variants.** A run that reached `done` is still a red gate if it measured fewer than it planned, so read the counts, not just the status.
- **A dry-run of a criterion stores nothing and is still a write.** Call it through `call_write_operation`.
- **Four verdicts, not two.** `stopped` is not `failed`: a case blocked by a guardrail or parked on an approval stopped on purpose. Read the per-case reason before you call anything a regression.
- **Run-versus-previous-run and variant-versus-variant are different reads.** The box serves the first. The second is a Studio screen, and there is no operation that returns it.

## What to tell the user

Say it before you launch anything, in two or three sentences, and never paste an operation id or a raw payload back at them.

- **What you are about to measure**, in their words: which cases, against what, and what a pass will mean.
- **Where they will see it**: the **Evals** screens for the cases, the suite and the run, and the run detail for the variant-by-variant table the box does not serve.
- **The one thing only they can decide**: whether a run may spend on a model, and which configuration column B points at.

Afterwards, one line: what it scored, which screen shows the verdicts, and what the next run would have to beat.

## What this skill does not cover

Testing the code around the model, which is an ordinary test suite against a real Postgres, not an eval. Model choice and pricing. Writing the agent or the workflow being measured.

## Other skills

- **stackbone**: the code inside the workspace, including a workflow used as a judge or as a persona.
- **stackbone-cli**: the rest of the CLI, including `agents list` for the installation id.
- **stackbone-debug**: a run that failed for a reason that is not the agent being wrong.
