# `stackbone db`

Two kinds of database verb live under `stackbone db`:

- **Migration verbs** (`migrate up` / `create` / `status`) are **drizzle-native** — they run locally against `STACKBONE_POSTGRES_URL` (the connection string `stackbone dev` exports; export it yourself otherwise). A missing URL exits `3` (`no_project`).
- **Explorer verbs** (`query` / `schemas` / `table`) are **read-only HTTP reads** against one installation through Studio — the local-dev install by default, or `--agent <id>`.

## Subcommands

| Command                                    | Description                                                                                                                    |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| `stackbone db migrate create <name>`       | Diff `src/schema.ts` against the journal and write a new SQL migration under `.stackbone/migrations/`.                         |
| `stackbone db migrate up [--target <tag>]` | Apply every pending migration safely with an advisory lock + journal. `--target <tag>` stops after that migration (inclusive). |
| `stackbone db migrate status`              | Classify each migration as `applied` / `pending` / `drifted`.                                                                  |
| `stackbone db query <sql>`                 | Run an ad-hoc **single SELECT** against the install. SQL from the positional, `--file <path>`, or stdin.                       |
| `stackbone db schemas`                     | List the schemas and tables visible to the install, with row estimates.                                                        |
| `stackbone db table <schema> <table>`      | Browse one table with cursor pagination (`--limit` 1-200 default 50, `--cursor`, `--order asc\|desc`).                         |

> There is **no** `db add-rag` command. The RAG schema is platform-provisioned per install (no creator step). Schema changes go through `db migrate create` — `db query` is **read-only** and the backend rejects anything that isn't a single SELECT.

## Target resolution

By default, every `db` subcommand targets the **agent connected to the active session**:

- If a `stackbone dev` session is running and `.stackbone/project.json.localDevInstallationId` is set → targets the local Postgres docker-compose container.
- Else if `--agent <id>` is passed → targets that cloud agent's Neon (read via Studio's gateway, write via the platform's migration runner).
- Else → exits with code `3` (`no_project`).

```sh
# Targets the active local-dev install
stackbone db migrate status

# Targets a specific cloud agent
stackbone db migrate status --agent agt_01HX...
```

## Migration files

Filenames are timestamped: `migrations/20260518091500_create-contracts.sql`. The numeric prefix sorts deterministically — newest file is the latest in lex order.

Inside the file: raw SQL. Drizzle's `pg-core` is also supported via a TypeScript shim (`migrations/20260518091500_create-contracts.ts`) if you prefer; the CLI auto-detects.

**Rules:**

- **Do not put `BEGIN` / `COMMIT` / `ROLLBACK`** — migrations run inside a backend-managed transaction with an advisory lock.
- **Never edit a file after it has been applied** to any environment. Create a new migration to amend.
- **Always commit migrations to git.** The `migrations/` directory is the source of truth; the container ships them and replays on startup.

## Subcommand: `migrate create`

```sh
stackbone db migrate create create-contracts
# Generates: migrations/20260518091500_create-contracts.sql
```

Skeleton content:

```sql
-- migrations/20260518091500_create-contracts.sql
CREATE TABLE contracts (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  title text NOT NULL,
  body text,
  created_at timestamptz DEFAULT now()
);

CREATE INDEX contracts_title_trgm_idx ON contracts USING gin (title gin_trgm_ops);
```

## Subcommand: `migrate up`

```sh
# Apply every pending migration in order
stackbone db migrate up --json

# Stop after applying the migration with this tag (inclusive)
stackbone db migrate up --target 20260518091500_create-contracts
```

There is no `--all` flag (applying all pending is the default) and no per-file positional — staged rollouts use `--target <tag>`.

**Idempotency.** Re-running after a partial failure resumes from the last successfully applied migration. The journal records every successful application; the JSON payload returns `{ applied, skipped }` with each entry's `tag` and `appliedAt`.

## Subcommand: `migrate status`

```sh
stackbone db migrate status --json
```

```json
{
  "ok": true,
  "data": {
    "applied": [
      { "version": "20260514091500", "name": "create-users", "applied_at": "2026-05-14T09:16:02Z" }
    ],
    "pending": [{ "version": "20260518091500", "name": "create-contracts" }]
  }
}
```

> **RAG schema is platform-provisioned.** There is no `db add-rag` command anymore — `rag_chunks`, the `pgvector` HNSW index and the async-ingest jobs table are created on every install by the platform migrator. `stackbone.rag` works without you migrating anything. Don't hand-write that schema.

## Subcommand: `query` (read-only)

```sh
# SQL as the positional (psql -c style)
stackbone db query "select id, status from runs order by created_at desc limit 20" --json

# Or from a file / stdin
stackbone db query --file ./report.sql --json
echo "select count(*) from users" | stackbone db query --json
```

The backend enforces **single SELECT only** — a write or multiple statements is rejected. Rows truncate at 1000 (`truncated: true` in the payload). The JSON payload is `{ columns, rows, truncated, duration_ms }`.

## Subcommand: `schemas`

```sh
stackbone db schemas --json   # { schemas: [{ name, tables: [{ name, estimated_rows }] }] }
```

## Subcommand: `table` (cursor pagination)

```sh
stackbone db table public users --limit 50 --json
stackbone db table public users --cursor "<nextCursor>" --order desc --json
```

Payload: `{ columns, rows, cursor_column, nextCursor, prevCursor }`. Walk pages with `nextCursor`.

## Exit codes

| Code | When                                                   |
| ---- | ------------------------------------------------------ |
| 0    | Success                                                |
| 1    | SQL error, migration error, parse failure              |
| 2    | Not authenticated                                      |
| 3    | No project linked / no target resolved                 |
| 5    | Permission denied (role lacks privilege on the target) |

## Common mistakes

- **Editing an applied migration file.** Always create a new one. The journal table compares filename + checksum; editing an applied file makes `migrate status` flag a checksum mismatch and refuse to proceed.
- **Forgetting to apply migrations in production after `publish`.** The wrapper applies them on the first invocation after the new image boots; if you have warm machines, they continue serving the old version until they recycle. The migration verbs run drizzle-native against `STACKBONE_POSTGRES_URL`, not via `--agent`; point that env var at the cloud agent's database and run `stackbone db migrate up` to force-apply.
- **Hand-writing the RAG `pgvector` schema.** Don't — `stackbone.rag`'s tables and indexes are platform-provisioned on every install with the operator/opclass paired correctly. Hand-rolling them drifts from what the SDK expects.
- **Changing the schema outside a migration.** Migrations are the source of truth — anything the schema history should describe belongs in `migrations/`, otherwise the next `publish` to a fresh install recreates the schema without your changes.
