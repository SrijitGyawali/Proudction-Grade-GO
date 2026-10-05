# 13 · Migrations workflow

## Tools

- **[tern](https://github.com/jackc/tern)**: migration tool by the author of
  pgx. Used both as a **CLI** (Taskfile) and as a **library** (`migrator.go`).

Install the CLI once:

```bash
go install github.com/jackc/tern/v2@latest
```

## Day-to-day workflow (local)

```bash
cd apps/backend

# 1. create a new numbered file
task migrations:new name=create_todos
#   → internal/database/migrations/002_create_todos.sql

# 2. write the up + down SQL in it

# 3. apply it to your local DB
export BOILERPLATE_DB_DSN="postgres://postgres:postgres@localhost:5432/boilerplate?sslmode=disable"
task migrations:up        # asks "Are you sure? [y/N]"

# 4. roll back if needed (tern CLI)
tern migrate -m ./internal/database/migrations --conn-string "$BOILERPLATE_DB_DSN" --destination -1
```

> The backend README mentions `task migrations:down`, but the Taskfile does not
> define it. Use the `tern migrate --destination -1` command above, or add the
> task yourself (good exercise!).

## In deployed environments

Nothing to do manually. On boot, when `PRIMARY.ENV != "local"`, `main.go` calls
`database.Migrate()`, which applies any new embedded migrations before the
server starts.

**Caution with many replicas:** if 3 pods start at the same time, all 3 try to
migrate. Tern takes a Postgres **advisory lock** during migration, so they
don't corrupt anything; they queue. For big teams, a common pattern is a
separate migration step in CI/CD (or a `cmd/migrate` binary run as a
Kubernetes Job) before rolling out the new pods.

## Writing safe migrations: the expand/contract pattern

Renaming `todos.title` to `todos.name` with zero downtime:

| Step | Migration | Code deploy |
|------|-----------|-------------|
| 1. Expand | `ADD COLUMN name TEXT` | — |
| 2. | — | write to **both** `title` and `name`, read `title` |
| 3. Backfill | `UPDATE todos SET name = title WHERE name IS NULL` | — |
| 4. | — | read `name` |
| 5. | `ALTER COLUMN name SET NOT NULL` | stop writing `title` |
| 6. Contract | `DROP COLUMN title` | — |

A one-step `RENAME COLUMN` breaks every old pod that is still running during
the deploy.

## Template for a new table

```sql
CREATE TABLE todos (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     TEXT NOT NULL,
    title       TEXT NOT NULL CHECK (char_length(title) BETWEEN 1 AND 200),
    completed   BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT unique_todos_title UNIQUE (user_id, title)
);

CREATE INDEX idx_todos_user_id ON todos (user_id);

---- create above / drop below ----

DROP TABLE IF EXISTS todos;
```

- `gen_random_uuid()` is built into Postgres 13+.
- Constraint names follow `unique_<table>_<column>`, which `sqlerr` understands.
- `TIMESTAMPTZ` (with time zone), not `TIMESTAMP`. Always store UTC instants.

### Keeping `updated_at` fresh

Either set it in every `UPDATE` query (`SET updated_at = now()`), or add a
trigger once:

```sql
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER todos_updated_at BEFORE UPDATE ON todos
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

Next: [14 · Background jobs](14-background-jobs.md)
