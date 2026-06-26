# `stackbone connectors`

List the available connectors (the curated catalog) **and** the connections that exist for the current account, with their **ids and names**. This is the discovery command you run before calling a connection from agent or workflow code with `stackbone.connection('<id or unique name>')`: it tells you which connectors exist, which actions/triggers each one exposes, and which connection id or name to target.

## Synopsis

```sh
stackbone connectors [--json]
```

## Behavior

1. Fetches the public connector **catalog** (`GET /v1/connectors`) — every connector with its `authKind` and the ids of its `actions` and `triggers`.
2. Fetches the **workspace connections** for the active organization (`GET /v1/organizations/:slug/connections`) — credential-free rows: `id`, `name`, `connectorId`, `healthStatus`.
3. Merges them: each catalog connector is printed with the connections that target it nested underneath.

The command is **authenticated** (it shows your account's connections), so it needs a session — exit `2` if you are not logged in.

Listing connections requires the **`connections:manage`** capability (org owner/admin). If your session is authenticated but lacks it, the command **does not fail**: it still prints the catalog and adds a `connections_unavailable` note (exit `0`), so the catalog stays useful.

## JSON output (`--json`)

```json
{
  "schema_version": 1,
  "connectors": [
    {
      "id": "telegram",
      "displayName": "Telegram",
      "authKind": "api_key",
      "actions": ["send-message", "send-photo"],
      "triggers": ["message-received"],
      "connections": [
        {
          "id": "3f2a1b2c-89ab-4cde-8123-456789abcdef",
          "name": "Support bot",
          "healthStatus": "active"
        },
        { "id": "a91c0d34-...", "name": "Sales bot", "healthStatus": "expiring" }
      ]
    },
    {
      "id": "gmail",
      "displayName": "Gmail",
      "authKind": "oauth2",
      "actions": ["send-email"],
      "triggers": ["email-received"],
      "connections": [{ "id": "7b10...", "name": "founders@", "healthStatus": "active" }]
    }
  ]
}
```

When connections could not be listed (no `connections:manage`), every `connections` array is empty and a top-level `"connections_unavailable": "<reason>"` is present.

`healthStatus` is one of `active | expiring | error | revoked`. The connection `id` is printed in full (both in `--json` and human mode) so it can be copied straight into a `stackbone.connection(...)` call.

## Using the output with the SDK

Inside an agent tool or a workflow step, address the connection by **id or unique name** (both are in the output above), then call the action as a method. The action methods are typed from the connector's schema, generated into `.stackbone/connect.d.ts` on `stackbone dev`:

```ts
import { stackbone } from '@stackbone/sdk';

// By unique name (readable) or by id (unambiguous):
await stackbone.connection('Support bot').sendMessage({ chatId, text });
await stackbone.connection('3f2a1b2c-89ab-4cde-8123-456789abcdef').sendMessage({ chatId, text });
```

- With **one** connection for a connector, the unique name or id resolves it directly.
- With **several** connections of the same connector, pass the specific id or unique name. Run `stackbone connectors` to pick one.

> Connector authoring and the dedicated `@stackbone/sdk/connect` surface (`connect()`, `withConnect()`, `connectHeaders()`) live in the **stackbone** skill.

## Flags

| Flag     | Description                                                                           |
| -------- | ------------------------------------------------------------------------------------- |
| `--json` | Emit the versioned envelope above instead of the human table. Use this from an agent. |

## Exit codes

| Code | When                                                                              |
| ---- | --------------------------------------------------------------------------------- |
| 0    | Listed (including the degraded catalog-only case when connections aren't visible) |
| 1    | Network / unexpected response from the control plane                              |
| 2    | Not authenticated — run `stackbone login`                                         |

## Common mistakes

- **Expecting credentials in the output.** Connections are credential-free by design — you get `id`, `name`, `connectorId` and `healthStatus`, never tokens or api keys.
- **Hardcoding a connection id in agent code.** Prefer the **unique name** passed to `stackbone.connection(...)` — it's readable and survives a re-connect; fall back to the id only when names collide.
- **Assuming a connector has a connection.** A connector in the catalog with an empty `connections` array means nothing is connected yet — connect one from the web settings (api_key form or the OAuth flow) before `stackbone.connection(...)` can use it.
