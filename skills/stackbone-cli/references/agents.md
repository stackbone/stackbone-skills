# `stackbone agents`

Discover the agent installations in your organization. This is the **target selector** for every other agent-runtime surface: the ids and slugs it lists are what you pass as `--agent <id>` to `runs`, `logs`, `secrets`, etc. Because it is the selector, `agents` itself takes **no** `--agent` flag — it resolves the active organization from your session and reads the installations directly. Both verbs are read-only (no `--yes`).

## Subcommands

| Command                            | Description                                                      |
| ---------------------------------- | ---------------------------------------------------------------- |
| `stackbone agents list`            | List every install in the org (including the local-dev install). |
| `stackbone agents get <agentSlug>` | Read one install by its slug.                                    |

```sh
stackbone agents list --json
# { items: [{ id, agentSlug, agentVersion, status, kind, ... }] }

stackbone agents get my-bot --json
# { agent: { id, agentSlug, agentVersion, status, kind, templateName, templateOwnerOrgSlug, localTunnelUrl, createdAt, updatedAt } }
```

The `id` is the **installation id** — that is the value you pass as `--agent <id>` everywhere else. The `agentSlug` is what `agents get` takes as its positional.

## Exit codes

| Code | When                                           |
| ---- | ---------------------------------------------- |
| 0    | Listed / fetched                               |
| 1    | Network / unexpected response, or missing slug |
| 2    | Not authenticated — run `stackbone login`      |
| 4    | No install with that slug in the active org    |
