# Contributing

Thanks for helping improve Stackbone Agent Skills. The flow is intentionally light: skills are Markdown files with YAML frontmatter, so a one-line fix is a one-line PR.

## Where the canonical source lives

This repo is **subtreed** into the Stackbone monorepo at `skills/` (see the ADR on subtreed public repos). The two copies are kept in sync via:

```bash
# Pull upstream changes into the monorepo (run from the monorepo root)
git subtree pull --prefix=skills \
  https://github.com/stackbone-ai/stackbone-skills.git main --squash
```

The convention is **edit upstream first**, then pull into the monorepo. Cross-cutting fixes made inside the monorepo that touch both platform code and the skills can be pushed back with `git subtree push`, but that is an exception, not the default.

## How to add or edit a skill

1. Pick the right place:
   - SDK use → `skills/stackbone/<module>/`
   - CLI command → `skills/stackbone-cli/references/`
   - Diagnostic flow → `skills/stackbone-debug/`
2. Skill files are Markdown with YAML frontmatter. See `AGENTS.md` for the shape.
3. Keep code examples runnable against the published `@stackbone/sdk` and `@stackbone/cli` versions. If you depend on an unreleased flag or method, hold the PR until the release lands.
4. Cross-link to sibling skills instead of duplicating content. `[See the stackbone-cli skill for build / publish commands.](../stackbone-cli/SKILL.md)`

## Style

- Tone: concise, declarative, second person ("Run this, expect that"). No marketing copy.
- Avoid telling Claude **why** Stackbone exists — it should know that from the description. Tell Claude **how to do the thing** the user asked for.
- Show the `{ data, error }` destructure pattern in every SDK example. Code that hides errors trains the agent to hide errors.
- Show `--json --yes` on CLI examples by default — the assumption is that an agent is reading the output.
- Prefer tables over prose when listing flags, exit codes, modules, error codes.

## Don't add a skill for...

- One-off recipes — those belong in the agent's own README or in `apps/wiki`.
- Internal-only flows (release scripts, monorepo CI, contributor tooling).
- Sales / pricing — that belongs on the marketing site, not in a skill an agent loads on every invocation.

## Tests

There is no automated test suite for the Markdown content. Manual checks before merging:

- [ ] The example you added runs against the current published SDK / CLI.
- [ ] Internal links resolve.
- [ ] `SKILL.md` frontmatter passes a YAML parser (no tabs, balanced quotes).
- [ ] The skill **name** matches the directory.
