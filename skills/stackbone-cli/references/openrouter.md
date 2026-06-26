# `stackbone openrouter`

Inspect the OpenRouter wiring of the targeted agent installation. Targets the local-dev install by default; override with `--agent <id>`. Both verbs are read-only (no `--yes`).

## Subcommands

| Command                       | Description                                                                                                                                                                      |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `stackbone openrouter get`    | The install's key info: `mode` (`managed` cloud sub-key vs `byo` env key), `base_url`, `public_id`, `status`, monthly `spend_limit_usd`. The bearer value is **never** returned. |
| `stackbone openrouter models` | The global OpenRouter model catalogue with per-million-token input/output pricing and a coarse type bucket.                                                                      |

```sh
stackbone openrouter get --json
# { key: { configured, mode, base_url, public_id, status, spend_limit_usd } }
stackbone openrouter models --json
# { items: [{ id, type, input_per_million_usd, output_per_million_usd }], fetched_at }
```

> Like `secrets`, the plaintext key reveal is deliberately out of scope — it is a Studio-only human action. `get` exposes mode/public-id/spend-cap/status, never the secret bearer.

The org's OpenRouter sub-key is minted per organization (managed mode), so `mode: managed` is the default; `byo` appears only when the install supplies its own key.

## Exit codes

| Code | When                                                        |
| ---- | ----------------------------------------------------------- |
| 0    | Success                                                     |
| 1    | Network / unexpected response                               |
| 2    | Not authenticated                                           |
| 3    | No install resolved — pass `--agent` or run `stackbone dev` |
| 4    | Install not found                                           |
