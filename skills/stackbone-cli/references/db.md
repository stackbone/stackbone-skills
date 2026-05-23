# `stackbone db`

Manage Drizzle migrations and inspect data on the agent's dedicated Neon Postgres (or the local Postgres `stackbone dev` boots in docker-compose).

## Subcommands

| Command                                            | Description                                                                                                                                    |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `stackbone db migrate create <name>`               | Generate a new timestamped migration file under `migrations/`.                                                                                 |
| `stackbone db migrate up [--to <version>] [--all]` | Apply pending migrations safely with an advisory lock and journal.                                                                             |
| `stackbone db migrate status`                      | List migrations: applied (with timestamps) vs pending.                                                                                         |
| `stackbone db studio`                              | Launch the embedded read-only DB explorer.                                                                                                     |
| `stackbone db add-rag --name <collection>`         | Declarative shortcut: generate the schema + indexes a new RAG collection needs (`_rag_documents`, `_rag_chunks` with `pgvector` + `tsvector`). |
| `stackbone db query "<sql>" [--writable]`          | Run a single SQL statement. Default is read-only; `--writable` lets through DDL/DML — destructive, requires `--yes` to skip confirmation.      |

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
# Apply all pending in order
stackbone db migrate up --all --json

# Apply up to a specific version
stackbone db migrate up --to 20260518091500

# Apply one explicit file
stackbone db migrate up 20260518091500_create-contracts.sql
```

**Idempotency.** Re-running after a partial failure resumes from the last successfully applied migration. The journal table `_migrations` records every successful application.

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

## Subcommand: `db studio`

```sh
stackbone db studio
# Opens http://localhost:4243 (Studio bottom panel embeds it; this is the standalone view)
```

- **Read-only by default.** Connection uses a `stackbone_viewer` role with `SET default_transaction_read_only = on` and `statement_timeout = 5s` enforced at the DB role level, not the UI.
- **Pagination via cursor.** Tables open to the first 50 rows, cursor-paginated.
- **SQL console.** Single statement only; the parser validates `SELECT` / `EXPLAIN` shape; the role enforces the rest.

To write from `db studio`, exit it and use a migration. There is no "edit row" affordance — that's a deliberate safety choice.

## Subcommand: `db add-rag`

```sh
stackbone db add-rag --name contracts
```

Generates a migration that creates the standard RAG schema for a new collection:

- `_rag_documents` (id, source_url, parsed_at, raw_text, metadata jsonb)
- `_rag_chunks` (id, document_id, content, embedding `vector(1536)`, search_tsv `tsvector`)
- Indexes: HNSW on `embedding` (`vector_cosine_ops`), GIN on `search_tsv`, B-tree on `document_id`

The collection is then addressable via `client.rag.from('contracts')` in the agent code.

## Subcommand: `db query`

```sh
# Read-only by default
stackbone db query "SELECT count(*) FROM contracts" --json

# Mutate (requires --writable and --yes for destructive ops)
stackbone db query "ALTER TABLE contracts ADD COLUMN status text" --writable --yes
```

Single-statement only. For multi-statement work, use a migration.

⚠️ **destructive** when `--writable` is set. The CLI asks for confirmation unless `--yes` is passed. Cross-org accidents are prevented by exit code `3` requiring a linked project.

## Exit codes

| Code | When                                                                                                                      |
| ---- | ------------------------------------------------------------------------------------------------------------------------- |
| 0    | Success                                                                                                                   |
| 1    | SQL error, migration error, parse failure                                                                                 |
| 2    | Not authenticated                                                                                                         |
| 3    | No project linked / no target resolved                                                                                    |
| 5    | Permission denied (role lacks privilege, e.g. trying to `db query --writable` against a cloud agent without `owner` role) |

## Common mistakes

- **Editing an applied migration file.** Always create a new one. The journal table compares filename + checksum; editing an applied file makes `migrate status` flag a checksum mismatch and refuse to proceed.
- **Forgetting to apply migrations in production after `publish`.** The wrapper applies them on the first invocation after the new image boots; if you have warm machines, they continue serving the old version until they recycle. Force with `stackbone db migrate up --all --agent <id>` against the cloud agent.
- **Hand-writing `pgvector` indexes with the wrong distance op.** A query using `<->` (cosine) against an index built with `<#>` (inner product) does a sequential scan and returns wrong order. Use `stackbone db add-rag` to get this right by default.
- **Using `db query --writable` for things a migration would do.** It works, but it leaves the schema in a state the `migrations/` history doesn't describe. The next `publish` to a fresh install will recreate the schema without your hand-edits.
