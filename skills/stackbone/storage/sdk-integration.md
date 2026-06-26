# `stackbone.storage` — SDK integration

`@aws-sdk/client-s3` v3 wrapped over a single physical bucket dedicated to the agent. R2 in production, MinIO in `stackbone dev`. Listing reads object metadata (key, size, last-modified, etag) straight from S3 — there is no separate metadata table to keep in sync.

`import { stackbone } from '@stackbone/sdk'`. Reach `stackbone.storage` from any tool's `execute()` or any workflow step — the platform injects the S3/R2 (MinIO in dev) credentials and the SDK reads them lazily. Pass a **logical** bucket to `.from(name)`; the SDK prefixes every key with `<agentId>/<logicalBucket>/` so logical buckets share one physical bucket without collisions.

```ts
const avatars = stackbone.storage.from('avatars'); // logical bucket
const reports = stackbone.storage.from('reports'); // same physical bucket, different prefix
```

## Upload

```ts
const key = `user/${userId}/avatar.png`;

const { data, error } = await stackbone.storage
  .from('avatars')
  .upload(key, file, { contentType: 'image/png' });

if (error) throw new Error(error.message);
// `data` is `{ key, etag? }` — `key` is the handle you store and reuse.
```

`upload(key, body, options?)`: `key` is the user-facing key you choose; `body` accepts `Uint8Array`, `Blob` or `string`. Options:

| Option        | Default | Notes                                                                        |
| ------------- | ------- | ---------------------------------------------------------------------------- |
| `contentType` | none    | Set it for anything a browser renders (`image/png`, `application/pdf`, ...). |
| `metadata`    | none    | Free-form `Record<string, string>` stored alongside the object.              |

### Persist the `key`, resolve the URL when serving

`upload` does **not** return a URL — the `key` is the only handle you need. It drives `download`, `remove` and signed URLs. When serving, derive the URL from the stored `key` with `getPublicUrl(key)` (public bucket) or `getSignedDownloadUrl(key)` (private bucket).

```ts
// `stackbone.database` is native Drizzle — no `{ data, error }` envelope, a failed
// query throws. Tables are the pgTable objects from src/schema.ts; helpers from
// '@stackbone/sdk/db'.
await stackbone.database.update(users).set({ avatarKey: data.key }).where(eq(users.id, userId));
```

## Download

```ts
const { data, error } = await stackbone.storage.from('reports').download(key);

if (error) throw new Error(`Report ${key} missing: ${error.message}`);

// `data` is `Blob` — convert with arrayBuffer() / stream() / text() as needed.
const buffer = Buffer.from(await data.arrayBuffer());
```

For large objects (>10 MB), prefer `getSignedDownloadUrl()` and let the client `fetch()` directly — round-tripping bytes through the agent burns time and memory.

## Signed URLs

```ts
// Read URL — share with the user's browser
const { data, error } = await stackbone.storage
  .from('reports')
  .getSignedDownloadUrl(key, { expiresIn: 3600 }); // 1 hour

if (error) throw new Error(error.message);
// data === { url: 'https://...?X-Amz-Signature=...', expiresAt: Date }
```

For uploads, ask for a presigned PUT URL with `getSignedUploadUrl(key, { expiresIn, contentType })`. The client uploads straight to R2 — bytes never touch the agent. Use it for files larger than a few hundred KB or browser-originated uploads. Passing `contentType` pins it into the signature, so the uploader must send a matching `Content-Type` header.

`expiresIn` is in seconds; defaults to 3600 (1 hour) if omitted.

## Remove

```ts
const { data, error } = await stackbone.storage.from('reports').remove(key);

if (error) throw new Error(error.message);
// data === { key }
```

`remove(key)` deletes a single object and echoes back the `key`. To delete several, loop and call `remove` per key, branching on each `error`.

## List

```ts
const { data, error } = await stackbone.storage
  .from('reports')
  .list({ prefix: 'user/123/', limit: 100 });

if (error) throw new Error(error.message);

for (const obj of data.objects) {
  // obj === { key, size, lastModified?, etag? }
}

if (data.nextCursor) {
  // Paginate with `list({ prefix, cursor: data.nextCursor })`
}
```

Options: `prefix`, `limit` (any positive integer), `cursor`. `list()` reads directly from S3, paginating with the continuation token surfaced as `nextCursor`.

## Patterns

### Hash-content key for deduplication

```ts
import { createHash } from 'node:crypto';

const hash = createHash('sha256').update(bytes).digest('hex');
const key = `sha256/${hash}`;

await stackbone.storage.from('uploads').upload(key, bytes, { contentType });
// Same content → same key. R2 overwrites with identical bytes, no extra cost.
```

Useful for avatars, document uploads, or anywhere the same file may arrive twice.

### Tie the object to a row, compensate on failure

There is no cross-service transaction between R2 and Postgres — compensate manually when one half fails. `stackbone.database` is native Drizzle: pass the pgTable object, `await` returns the typed rows, and a failed query throws (no envelope), so wrap it in try/catch and roll the upload back.

```ts
const key = `contracts/${crypto.randomUUID()}.pdf`;
const upload = await stackbone.storage.from('contracts').upload(key, file, { contentType });
if (upload.error) throw new Error(upload.error.message);

try {
  const [contract] = await stackbone.database
    .insert(contracts)
    .values({ title, fileKey: upload.data.key })
    .returning();

  return { contract };
} catch (err) {
  await stackbone.storage.from('contracts').remove(upload.data.key); // orphaned objects burn storage
  throw err;
}
```

## Best practices

1. **Always destructure `{ data, error }`.** Same rule as every SDK storage call.
2. **Store the `key`.** It is the only handle you need; resolve the serving URL on demand with `getPublicUrl` / `getSignedDownloadUrl`.
3. **`remove` takes a single key.** Delete one at a time; loop for fan-out.
4. **Signed URL for >1 MB.** Don't round-trip large bytes through the agent; hand out `getSignedUploadUrl` / `getSignedDownloadUrl`.
5. **Use `metadata` for tags.** `{ ownerId, contentHash, version }` — stored alongside the object on `upload`.
6. **List through `stackbone.storage`.** Use `stackbone.storage.from(bucket).list()` rather than poking at object storage directly.
7. **Match logical bucket to access pattern.** `'public-avatars'`, `'private-reports'`, `'tmp-uploads'`. Public/private wiring lives on the physical R2 bucket — pick the right one per use case.

## Common mistakes

| Mistake                                                                             | Fix                                                                                                        |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `upload(file, { contentType })` (body first)                                        | `upload(key, file, { contentType })` — the key is the first argument.                                      |
| Expecting `url` back from `upload`                                                  | `upload` returns `{ key, etag? }`. Build the URL with `getPublicUrl(key)` / `getSignedDownloadUrl(key)`.   |
| `remove([key])` (array)                                                             | `remove(key)` — a single string key. Loop for multiple deletes.                                            |
| Round-tripping a 50 MB video through the agent                                      | Use `getSignedUploadUrl(key, { contentType })` for upload, `getSignedDownloadUrl(key)` for download.       |
| Passing a Node `Readable` / `ReadableStream` as the body                            | Body must be `Uint8Array`, `Blob` or `string`. Buffer the stream first, or hand out a presigned PUT URL.   |
| Forgetting `contentType: 'image/png'` and seeing the browser download a `.bin` file | Always set `contentType` for anything a browser renders.                                                   |
| Hard-coding the bucket env var                                                      | Don't. `stackbone.storage.from('logical-name')` is the only API; bucket configuration is platform-managed. |

## Common error codes

| `error.code`             | Cause                                                                | Action                                                                         |
| ------------------------ | -------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `s3_error`               | S3/R2 rejected the operation (missing key, ACL, size, network)       | Inspect `error.meta` (`httpStatusCode`, `fault`); retry only transient faults. |
| `s3_invalid_key`         | Key contains path-traversal segments (`..`)                          | Sanitize keys; keep them to plain `[A-Za-z0-9/_.-]` paths.                     |
| `s3_invalid_argument`    | `list({ limit })` with a non-positive `limit`                        | Pass a positive integer for `limit`.                                           |
| `s3_empty_response`      | `download` got an object with no body                                | Treat as missing/corrupt; do not retry blindly.                                |
| `s3_credentials_missing` | `STACKBONE_S3_ACCESS_KEY` / `_SECRET_KEY` / `_ENDPOINT` not injected | Configuration bug — surface; not retryable.                                    |
| `s3_bucket_missing`      | `STACKBONE_S3_BUCKET` not injected                                   | Configuration bug — surface; not retryable.                                    |
| `agent_id_missing`       | `STACKBONE_AGENT_ID` not injected                                    | Configuration bug — surface; not retryable.                                    |

See the main [SKILL.md](../SKILL.md) for the cross-module patterns and [agent-yaml.md](../agent-yaml.md) for the real (small) manifest surface. There is no `capabilities:` field — module access is negotiated through the protocol contract, not declared.
