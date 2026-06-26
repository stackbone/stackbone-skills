# `stackbone logs`

Live-tail the targeted agent installation's logs over a Server-Sent Events stream. One verb: `tail`. Targets the local-dev install by default; override with `--agent <id>`. Read-only (no `--yes`).

## `stackbone logs tail`

```sh
stackbone logs tail --json                       # last 100 events, then stop
stackbone logs tail --follow --json              # stream until Ctrl-C
stackbone logs tail --run run_01HX... --json     # restrict to one run (per-run endpoint)
stackbone logs tail --level error --q timeout --json
```

| Flag         | Where applied   | Description                                                                           |
| ------------ | --------------- | ------------------------------------------------------------------------------------- |
| `--run`      | server-side     | Restrict to logs of one run id (uses the per-run endpoint; do not also need a query). |
| `--level`    | server-side     | Minimum Pino level (`info`, `warn`, `error`, …).                                      |
| `--q`        | server-side     | Free-text substring over each line.                                                   |
| `--trace-id` | server-side     | Restrict to one trace id.                                                             |
| `--since`    | **client-side** | Drop events older than a duration (`15m`, `2h`, `30s`, `1d`) or an ISO timestamp.     |
| `--until`    | **client-side** | Drop events newer than a duration or ISO timestamp.                                   |
| `--follow`   | —               | Keep the stream open until Ctrl-C. Without it, stop after `--limit` events.           |
| `--limit`    | **client-side** | Max events to print when **not** following (default 100).                             |

`--since` / `--until` / `--limit` have no server-side equivalent on this stream, so the CLI filters them client-side: it compares each frame's `time` and stops once `--limit` is reached (unless `--follow`). With `--json`, each printed event is one JSON envelope per line (a `StudioLogEvent`).

## Exit codes

| Code | When                                                               |
| ---- | ------------------------------------------------------------------ |
| 0    | Stream ended cleanly (limit reached, or Ctrl-C while following)    |
| 1    | Network / stream error, or a bad `--limit` / `--since` / `--until` |
| 2    | Not authenticated                                                  |
| 3    | No install resolved — pass `--agent` or run `stackbone dev`        |
