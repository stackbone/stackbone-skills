# `stackbone publish`

Package the current workspace so it can be deployed. Discovered by convention — every `deep-agents/<name>/` folder with an `index.ts` and every `workflows/<name>.workflow.ts` — `publish` **esbuilds every deep agent** (to `deep-agents/<name>/index.mjs`, with `@stackbone/sdk`, `deepagents` and `@langchain/*` kept **external** so the runtime image resolves one copy) + every workflow on this host and packs them into a tar, written locally. A `stackbone.config.ts` can optionally override the workflow list, but most workspaces don't have one.

Native dependencies (`.node` add-ons) are rejected up-front — only pure JS runs in the runtime image.

## Synopsis

```sh
stackbone publish [--json]
```

> There is **no** `--version`, `--dry-run`, `--yes`, `--verbose` or `--cache` on `publish`, and no Trivy/cosign/registry-push step. Those were part of an older container-publish flow that no longer exists.

## What it does

1. Discover the workspace by convention (every `deep-agents/<name>/index.ts` + every `workflows/<name>.workflow.ts`), then esbuild each agent + build each workflow on this host. The bundle keeps `@stackbone/sdk` + `deepagents` + `@langchain/*` external so each agent shares one invocation context and one LangChain copy; an inlined SDK or a native dep aborts with an actionable message.
2. Pack the outputs into `dist/stackbone/workspace-bundle.tar`.
3. Write a `dist/stackbone/workspace-bundle.json` pointer beside it with the digest, byte sizes, the agent + workflow names, the tar path and the manifest.
4. Print the digest. The upload to object storage + the build-pointer registration are handled by the platform's provisioning flow on deploy — the CLI produces the verifiable artifact, it does not push it.

```sh
stackbone publish
# Agents:    support, billing
# Workflows: onboarding
# Digest:    sha256:<…>
# Size:      …
# Bundle:    dist/stackbone/workspace-bundle.tar

stackbone publish --json
# { schema_version, kind: "workspace", digest, sizeBytes, agents, workflows, tar }
```

## Exit codes

| Code | When                                         |
| ---- | -------------------------------------------- |
| 0    | Bundle written                               |
| 1    | Native dep detected, or a build/bundle error |

## Common mistakes

- **Looking for `--version`.** A workspace bundle is identified by its content digest, not a flag.
- **Shipping a native dependency.** Anything with a `.node` binding is rejected before bundling — only pure JS runs in the runtime image.
- **Inlining `@stackbone/sdk`.** The build keeps the SDK external on purpose so each agent shares one invocation context; bundling it in aborts the publish.
