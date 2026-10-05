# `internal/database/` — PostgreSQL connection pool + migrations

## Job of this package

1. Open **one shared connection pool** to Postgres when the app starts.
2. Attach **tracing/logging** to every query.
3. Apply **schema migrations** (SQL files) in order.

## Files

| File                 | What it does |
|----------------------|--------------|
| `database.go`        | `Database` struct (wraps `*pgxpool.Pool`), `New()` builds the DSN, configures tracing, creates the pool and pings it; `Close()` |
| `migrator.go`        | `Migrate()` runs every `migrations/*.sql` file not yet applied, using **tern** |
| `migrations/`        | Numbered SQL files, embedded into the binary (see its own README) |

## `database.go` in detail

### Building the DSN

```go
postgres://USER:PASSWORD@HOST:PORT/NAME?sslmode=SSL_MODE
```

- `net.JoinHostPort` handles IPv6 hosts correctly (`[::1]:5432`).
- `url.QueryEscape(password)` stops passwords with `@`, `/` or `:` from
  breaking the URL.

### Why a pool (`pgxpool`) and not a single connection?

A web server handles many requests **at the same time**. One connection can
only run one query at a time. A pool keeps several open connections and lends
one to each request, then takes it back. Opening a TCP + TLS + auth connection
per request would be very slow.

### Query tracing

```
env == "local"                → log every SQL statement (pretty console)
New Relic license key present → nrpgx5 tracer (DB timing in APM)
both                          → multiTracer runs both
```

`multiTracer` is a small adapter. pgx accepts only **one** `Tracer`, so this
type holds several and forwards `TraceQueryStart`/`TraceQueryEnd` to each. It
is a good example of the *composite pattern* in Go.

### Ping on startup

`pool.Ping()` with a 10 s timeout. If the database is unreachable the app
**refuses to start**, so you never get a "running" server that cannot do
anything.

## Pool-size settings (from config)

`DatabaseConfig` defines `MaxOpenConns`, `MaxIdleConns`, `ConnMaxLifetime` and
`ConnMaxIdleTime`, but `database.New()` **does not apply them yet**. The pool
uses pgxpool defaults (max conns = max(4, number of CPUs)). To use them:

```go
pgxPoolConfig.MaxConns          = int32(cfg.Database.MaxOpenConns)
pgxPoolConfig.MinConns          = int32(cfg.Database.MaxIdleConns)
pgxPoolConfig.MaxConnLifetime   = time.Duration(cfg.Database.ConnMaxLifetime) * time.Second
pgxPoolConfig.MaxConnIdleTime   = time.Duration(cfg.Database.ConnMaxIdleTime) * time.Second
```

This would make a good first contribution. See [`../../docs/12-database-pooling.md`](../../docs/12-database-pooling.md).

## `migrator.go` in detail

- `//go:embed migrations/*.sql` **bakes the SQL files into the compiled
  binary**. You deploy one file, and the migrations travel with it.
- `tern.NewMigrator(ctx, conn, "schema_version")` stores the current version
  number in a table called `schema_version`.
- It compares *current version* vs *number of files*, applies the missing ones
  in order, and logs `migrated database schema, from X to Y`.
- It uses a **single** `pgx.Conn` (not the pool) because migrations are a
  one-off, sequential job.

## Who uses this package?

- `server.New()` calls `database.New()` and stores it as `Server.DB`.
- `main.go` calls `database.Migrate()` (non-local envs).
- `internal/testing` calls both against a Docker Postgres.
- Repositories use `server.DB.Pool` to run queries.
