# `client.storage` — SDK integration

`@aws-sdk/client-s3` v3 wrapped over a single bucket dedicated to the agent. R2 in production, MinIO in `stackbone dev`. Object metadata (key, size, content type, etag, owner) lives in the agent's Neon in the `_storage_objects` table — read it through `client.storage`, never with raw SQL.

## Connection

```ts
import { createClient } from '@stackbone/sdk';

const client = createClient(); // reads AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / S3_ENDPOINT
```

The platform injects `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `S3_ENDPOINT` at boot. The bucket name is fixed per agent — pass a **logical bucket** to `.from(name)`; the SDK prefixes the physical key with `<agentId>/<logicalBucket>/` so multiple logical buckets share the same physical R2 bucket without collisions.

```ts
const avatars = client.storage.from('avatars'); // logical bucket
const reports = client.storage.from('reports'); // same physical bucket, different prefix
```

## Upload

```ts
const { data, error } = await client.storage
  .from('avatars')
  .upload(file, { contentType: 'image/png' });

if (error) return ctx.fail('storage_upload_failed', error.message);

// `data` is `{ key, url, etag, size }` — `url` is the public URL (if the bucket
// is public), `key` is the opaque identifier you store in DB.
```

`file` accepts `Uint8Array`, `Buffer`, `Blob`, `ReadableStream` or a Node `Readable`. Options:

| Option         | Default                      | Notes                                                                     |
| -------------- | ---------------------------- | ------------------------------------------------------------------------- |
| `key`          | `crypto.randomUUID()`        | Pass your own when you need predictable paths (`user/<id>/avatar.png`).   |
| `contentType`  | `'application/octet-stream'` | Required for browsers to render images / play media correctly.            |
| `cacheControl` | none                         | `'public, max-age=31536000, immutable'` for content-hashed assets.        |
| `metadata`     | none                         | Free-form `Record<string, string>` returned by `list()` and `download()`. |

### Persist both `url` and `key`

```ts
const { data, error } = await client.storage
  .from('avatars')
  .upload(file, { contentType: 'image/png' });

if (error) return ctx.fail('storage_upload_failed', error.message);

await client.database
  .from(users)
  .update({ avatarUrl: data.url, avatarKey: data.key })
  .where(eq(users.id, userId));
```

`url` is for serving; `key` is for everything else (`remove`, `signedUrl`, `download`). Losing the `key` means losing the ability to delete or move the object — always store both.

## Download

```ts
const { data, error } = await client.storage.from('reports').download(key);

if (error) return ctx.fail('storage_not_found', `Report ${key} missing`);

// `data` is `Blob` — convert with arrayBuffer() / stream() / text() as needed.
const buffer = Buffer.from(await data.arrayBuffer());
```

For large objects (>10 MB), prefer `signedUrl()` and let the client `fetch()` directly — round-tripping bytes through the agent burns invocation time and memory.

## Signed URLs

```ts
// Read URL — share with the user's browser
const { data, error } = await client.storage.from('reports').signedUrl(key, 3600); // expires in 1 hour

if (error) return ctx.fail('storage_signed_url_failed', error.message);
// data === { url: 'https://...?X-Amz-Signature=...', expiresAt: Date }
```

For uploads, ask for a presigned PUT URL with `signedUrl(key, expiresIn, { method: 'PUT', contentType })`. The client uploads straight to R2 — bytes never touch the agent. Use this for files larger than a few hundred KB or when the upload originates in the browser.

`expiresIn` is in seconds. Defaults to 900 (15 min) if omitted. Hard cap is 7 days (R2 limit, not Stackbone).

## Remove

```ts
const { data, error } = await client.storage.from('reports').remove([key1, key2, key3]); // always an array, even for a single key

if (error) return ctx.fail('storage_delete_failed', error.message);
// data === { deleted: 3 }
```

`remove` mirrors the database `insert` convention: **the argument is always an array**, even for one key. This forces batched semantics for fan-out and keeps single-key deletes explicit. Passing a bare string is rejected.

## List

```ts
const { data, error } = await client.storage
  .from('reports')
  .list({ prefix: 'user/123/', limit: 100 });

if (error) return ctx.fail('storage_list_failed', error.message);

for (const obj of data.objects) {
  // obj === { key, size, lastModified, etag, contentType }
}

if (data.nextCursor) {
  // Paginate with `list({ prefix, cursor: data.nextCursor })`
}
```

Options: `prefix`, `limit` (default 1000, max 1000), `cursor`. The `_storage_objects` table is the source of truth — `list()` does not round-trip to R2.

## Patterns

### Hash-content key for deduplication

```ts
import { createHash } from 'node:crypto';

const hash = createHash('sha256').update(bytes).digest('hex');
const key = `sha256/${hash}`;

const { data, error } = await client.storage.from('uploads').upload(bytes, { key, contentType });

// Same content → same key. R2 overwrites with identical bytes, no extra storage cost.
```

Useful for user avatars, document uploads, or anywhere the same file may arrive twice.

### Upload from a stream

```ts
import { Readable } from 'node:stream';

const stream = Readable.fromWeb(response.body); // body from fetch()
const { data, error } = await client.storage
  .from('imports')
  .upload(stream, { key: `csv/${Date.now()}.csv`, contentType: 'text/csv' });
```

Streams keep memory flat for large objects. Combine with `signedUrl` PUT when the client uploads directly.

### Tie the object to a row before returning

```ts
const upload = await client.storage.from('contracts').upload(file, { contentType });
if (upload.error) return ctx.fail('storage_upload_failed', upload.error.message);

const insert = await client.database
  .from(contracts)
  .insert([{ title, fileKey: upload.data.key, fileUrl: upload.data.url }])
  .returning();

if (insert.error) {
  // Roll the upload back — orphaned objects burn storage cost
  await client.storage.from('contracts').remove([upload.data.key]);
  return ctx.fail('db_insert_failed', insert.error.message);
}

return ctx.ok({ contract: insert.data[0] });
```

There is no cross-service transaction between R2 and Postgres. Compensate manually when one half fails.

## Best practices

1. **Always destructure `{ data, error }`.** Same rule as every SDK module.
2. **Store both `url` and `key`.** `url` for serving, `key` for management.
3. **`remove` takes an array.** Single-key deletes are `remove([key])`, never `remove(key)`.
4. **Stream or signed URL for >1 MB.** Don't round-trip large bytes through `invoke`.
5. **Use `metadata` for searchable tags.** `{ ownerId, contentHash, version }` — surfaced by `list()`.
6. **Don't read `_storage_objects` with `client.database`.** Use `client.storage.list()`. The table shape is internal and may change.
7. **Match logical bucket to access pattern.** `'public-avatars'`, `'private-reports'`, `'tmp-uploads'`. Public/private wiring lives on the physical R2 bucket — pick the right one per use case.

## Common mistakes

| Mistake                                                                             | Fix                                                                                                     |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `remove(key)` (string)                                                              | `remove([key])` (array)                                                                                 |
| Storing only `url`, then trying to delete                                           | Store `key` too. The `url` is not reversible to a `key` after the fact.                                 |
| Round-tripping a 50 MB video through `invoke`                                       | Use `signedUrl(..., expiresIn, { method: 'PUT' })` for upload, `signedUrl()` for download.              |
| `client.database.from('_storage_objects').select()`                                 | `client.storage.from(bucket).list(...)` — internal table, do not query directly.                        |
| Forgetting `contentType: 'image/png'` and seeing the browser download a `.bin` file | Always set `contentType` for anything a browser renders.                                                |
| Hard-coding the bucket env var                                                      | Don't. `client.storage.from('logical-name')` is the only API; bucket configuration is platform-managed. |

## Common error codes

| `error.code`                 | Cause                                                      | Action                                                             |
| ---------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------ |
| `storage_not_found`          | `download` / `remove` on a missing key                     | Surface `not_found`; do not retry.                                 |
| `storage_signed_url_expired` | Client used a presigned URL past `expiresAt`               | Issue a new URL — old ones are not revivable.                      |
| `storage_upload_failed`      | R2 rejected the PUT (size, ACL, network)                   | Inspect `error.details`; retry only if `error.retryable === true`. |
| `storage_quota_exceeded`     | Org tier storage cap hit                                   | Surface verbatim — the org member must upgrade. Do not retry.      |
| `storage_invalid_key`        | Key contains `..`, leading `/`, or non-ASCII control chars | Sanitize keys; keep them to `[A-Za-z0-9/_.-]`.                     |

See the main [SKILL.md](../SKILL.md) for the cross-module patterns and [agent-yaml.md](../agent-yaml.md) for declaring `capabilities: [storage]` (auto-declared whenever any `client.storage.*` call is in the bundle).
