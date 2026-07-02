# `stackbone login`

Authenticate the CLI against the Stackbone control plane using OAuth 2.0 device flow (RFC 8628).

## Synopsis

```sh
stackbone login [--json] [--verbose]
```

## Behavior

1. The CLI requests a device code from the platform and tries to open the user's default browser at the code-entry URL.
2. If the browser can't be opened (no display, SSH session, container shell), the CLI **prints the URL and code in plain text** — copy-paste them into any browser on any machine.
3. The CLI polls the platform until the code is approved or rejected.
4. On approval, credentials are written to `~/.stackbone/credentials.json` with mode `0600`.

## Why device flow

Device flow lets a CLI on a headless machine, a CI runner, or a dev container authenticate via a browser on a totally different device. No localhost callback server, no PKCE redirect URI to register, no client secrets. The trade-off is one extra step for the user (code entry) — acceptable for an inner-loop CLI.

## Flags

| Flag        | Description                                                                                                                               |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `--json`    | Emit envelope on success / failure. The flow itself still needs the user to approve in a browser; `--json` only affects the CLI's output. |
| `--verbose` | Print the polling cycle (every ~5 s) so the user can see the CLI is alive.                                                                |

## Examples

```sh
# Interactive, opens browser
stackbone login

# No browser on this machine: print the URL + code to enter from any device
stackbone login --no-browser
```

## CI / agent shell

There is **no token env var and no PAT** — authentication comes only from `~/.stackbone/credentials.json`, minted by `stackbone login`. For non-interactive contexts, run `stackbone login` once on a machine with a browser and carry that file into the runner (it holds a refresh token, so it keeps working without re-login). Verify with `stackbone whoami --json`.

## What's stored

`~/.stackbone/credentials.json`:

```json
{
  "accessToken": "stk_at_...",
  "refreshToken": "stk_rt_...",
  "expiresAt": "2026-06-20T08:30:00.000Z",
  "user": {
    "id": "usr_...",
    "email": "..."
  }
}
```

- `chmod 600` on the file.
- The refresh token is used automatically by every command — you do not need to re-run `login` daily.
- Logout (`stackbone logout`) removes the file.

## Exit codes

| Code | When                                       |
| ---- | ------------------------------------------ |
| 0    | Logged in (or already logged in)           |
| 1    | Network error reaching the platform        |
| 2    | Device code rejected / expired by the user |

## Common mistakes

- **Running `stackbone login` in a script.** The script blocks waiting for browser approval. Pre-seed `~/.stackbone/credentials.json` from an interactive login instead — there is no token env var.
- **Committing `~/.stackbone/credentials.json`.** It's chmod 600 and outside the project, but worth noting — never copy it into the project tree.
