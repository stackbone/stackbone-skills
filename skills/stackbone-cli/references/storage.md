# `stackbone storage`

Operate the S3-style object store bound to the targeted agent installation (R2 in cloud, MinIO locally). Targets the local-dev install by default; override with `--agent <id>`. Every object verb needs a `--bucket` (an install can expose more than one). `list` is paginated (`--limit` + `--cursor`). `remove` is **destructive** and requires `--yes`.

## Subcommands

| Command                                                  | Description                                                                                                                  |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `stackbone storage buckets`                              | List the buckets the install exposes.                                                                                        |
| `stackbone storage list --bucket <b>`                    | List objects, optionally under `--prefix`. `--limit` (1-1000, default 100), `--cursor`. Returns `items` + `common_prefixes`. |
| `stackbone storage get <key> --bucket <b>`               | Download via a short-lived presigned URL to `--out <path>` (else stdout). There is no direct GET-object route.               |
| `stackbone storage put <key> --bucket <b> --file <path>` | Upload a local file under the key (multipart).                                                                               |
| `stackbone storage presign <key> --bucket <b>`           | Print a short-lived presigned download URL + its `expires_at`.                                                               |
| `stackbone storage remove <key> --bucket <b>` ✱          | Delete an object. Requires `--yes`.                                                                                          |

```sh
stackbone storage buckets --json
stackbone storage list --bucket uploads --prefix images/ --json
stackbone storage get reports/q3.pdf --bucket uploads --out ./q3.pdf --json
stackbone storage put logo.png --bucket uploads --file ./logo.png --json
stackbone storage remove old.txt --bucket uploads --yes --json
```

The backend reserves the `rag/` key prefix — `put`/`remove` under it are rejected (use the `rag` surface). When downloading to stdout the `--json` envelope only confirms where the bytes landed, so a piped consumer never sees the body interleaved.

## Exit codes

| Code | When                                                                   |
| ---- | ---------------------------------------------------------------------- |
| 0    | Success                                                                |
| 1    | Network / validation (missing `--bucket`, blank key, missing `--file`) |
| 2    | Not authenticated                                                      |
| 3    | No install resolved — pass `--agent` or run `stackbone dev`            |
| 4    | Bucket or object not found                                             |
| 5    | `remove` without `--yes`                                               |
