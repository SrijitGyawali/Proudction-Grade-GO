# `internal/database/migrations/` — versioned schema changes

## What is a migration?

A migration is a **small, numbered SQL file** that changes the database schema
one step at a time: create a table, add a column, add an index.

Instead of "the schema is whatever someone typed into psql", the schema becomes
**code**: reviewed in PRs, versioned in git, and applied the same way on every
laptop, CI run and production server.

## File format (tern)

```sql
-- 002_create_todos.sql

CREATE TABLE todos (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title       TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

---- create above / drop below ----

DROP TABLE todos;
```

- Everything **above** the separator line is the **up** migration.
- Everything **below** is the **down** (rollback) migration.
- `001_setup.sql` is an empty placeholder that shows the format.

## Naming

`NNN_description.sql`. The number decides the order. Tern refuses gaps or
duplicates, which protects you from two people both creating `002_…`.

Create one with:

```bash
task migrations:new name=create_todos      # runs: tern new -m ./internal/database/migrations create_todos
```

## How they get applied

| Where            | How |
|------------------|-----|
| Local dev        | `task migrations:up` (asks for confirmation, uses `BOILERPLATE_DB_DSN`) |
| Staging / prod   | Automatically on boot: `main.go` → `database.Migrate()` (because `env != "local"`) |
| Tests            | `internal/testing.SetupTestDB()` runs `database.Migrate()` on a fresh container |

The files are compiled into the binary with `//go:embed`, so production does
not need this folder on disk.

## Rules for production-grade migrations

1. **Never edit a migration that has already run anywhere shared.** Write a
   new one instead. Tern tracks version *numbers*, not file contents.
2. **Make changes backwards compatible.** While a deploy is rolling out, old
   and new code run at the same time. Add a nullable column first, deploy code
   that writes it, then make it `NOT NULL` in a later migration.
3. **Name your constraints** (`CONSTRAINT unique_users_email UNIQUE (email)`).
   `internal/sqlerr` reads constraint names to build friendly error messages.
4. **Add indexes for every foreign key and every column you filter by.**
5. Use `CREATE INDEX CONCURRENTLY` on big tables. A tern file that contains it
   must not run inside a transaction, so check tern's docs.
6. Always add `created_at` / `updated_at`. `model.Base` expects them.

See [`../../../docs/13-migrations-workflow.md`](../../../docs/13-migrations-workflow.md).
