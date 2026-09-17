# Contributing

Thanks for helping improve Stackbone Agent Skills. The flow is intentionally light: skills are Markdown files with YAML frontmatter, so a one-line fix is a one-line PR.

## Where the canonical source lives

This repo is **subtreed** into the Stackbone monorepo at `skills/` (see the ADR on subtreed public repos). The two copies are kept in sync via:

```bash
# Pull upstream changes into the monorepo (run from the monorepo root)
git subtree pull --prefix=skills \
  https://github.com/stackbone/stackbone-skills.git main --squash
```

The convention is **edit upstream first**, then pull into the monorepo. Cross-cutting fixes made inside the monorepo that touch both platform code and the skills can be pushed back with `git subtree push`, but that is an exception, not the default.

## Skills hold procedure; the docs hold facts

Before you write, decide which of the two you are changing:

| You want to…                                                    | Edit                                                                                        |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Change how an agent should proceed (order, rule, pitfall, loop) | The `SKILL.md` of the skill that owns it                                                    |
| Make a new or retitled docs page reachable                      | The finding rules of the owning `SKILL.md` (usually nothing: the rules reach pages by name) |
| Change an interview question                                    | `skills/stackbone-coder/references/*.md`                                                    |
| Document a flag, a method, an option, an error code, a limit    | The public docs (`apps/wiki` in the Stackbone monorepo), not a skill                        |

A PR that adds a flag table, a method list or a catalogue of pages to a skill will be asked to move the facts to the docs and keep only the rule that finds them. `AGENTS.md` explains why.

## How to edit a skill

1. Skill files are Markdown with YAML frontmatter. See `AGENTS.md` for the shape and the three blocks every `SKILL.md` carries.
2. Keep code blocks to the minimal shape. Worked examples live under Examples on the docs site.
3. Keep code examples runnable against the published `@stackbone/sdk` and `@stackbone/cli` versions. If you depend on an unreleased flag or method, hold the PR until the release lands.
4. Cross-link sibling skills by name instead of duplicating content: `use the stackbone-cli skill for dev / build`.
5. Bump the version in `.claude-plugin/plugin.json` and in each `SKILL.md` you touched (same number everywhere).

## Style

- Tone: concise, declarative, second person ("Run this, expect that"). No marketing copy.
- Avoid telling Claude **why** Stackbone exists — it should know that from the description. Tell Claude **how to do the thing** the user asked for, and **where to read** the rest.
- Show the `{ data, error }` destructure pattern in every SDK example. Code that hides errors trains the agent to hide errors.
- Show `--json --yes` on CLI examples by default — the assumption is that an agent is reading the output.
- Name docs pages by their title and place as `list_docs` shows them (`` `stackbone.rag` (SDK › Data) ``), never by a path: paths move, titles stay.

## Don't add a skill for...

- One-off recipes — those belong in the agent's own README or in the docs' examples section.
- Internal-only flows (release scripts, monorepo CI, contributor tooling).
- Sales / pricing — that belongs on the marketing site, not in a skill an agent loads on every invocation.

## Tests

There is no automated test suite for the Markdown content. Manual checks before merging:

- [ ] Every page you named exists under that title in `list_docs` on `https://docs.stackbone.ai/mcp` (or `https://docs.stackbone.ai/llms.txt`).
- [ ] The example you added runs against the current published SDK / CLI.
- [ ] Internal links resolve.
- [ ] `SKILL.md` frontmatter passes a YAML parser (no tabs, balanced quotes).
- [ ] The skill **name** matches the directory.
