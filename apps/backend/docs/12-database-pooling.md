# 12 · Database access and connection pooling

## Why pgx (and not `database/sql` + `lib/pq`, or an ORM)?

- **pgx** is the most complete, fastest pure-Go Postgres driver.
- Native support for Postgres types (UUID, JSONB, arrays, timestamptz).
- `pgxpool` is a built-in, battle-tested pool.
- Helpers like `pgx.CollectRows` + `RowToStructByName` give you ORM-like
  scanning while you still write **plain SQL**.

Writing SQL by hand is common in production Go. You see exactly what runs, and
you can tune it with `EXPLAIN ANALYZE`. If you want type-safety on top, look at
**sqlc** (it generates Go code from `.sql` files).

## How a pool works

```
 requests        pool (max N)                      Postgres
 ─────────     ┌────────────────┐
 req A ───────►│ conn 1 (busy)  │──────────────────►
 req B ───────►│ conn 2 (busy)  │──────────────────►
 req C ───────►│ conn 3 (idle)  │  ← reused, no new handshake
 req D  waits  │                │
               └────────────────┘
```

- Opening a connection costs TCP + TLS + auth (several ms). Reuse is much faster.
- **Postgres has a hard connection limit** (often 100). With 5 replicas × 25
  conns = 125, you're over. **Size pools per replica**:
  `max_conns_per_pod ≈ (db_max_connections − headroom) / replicas`.
- When all connections are busy, new requests **wait**. With a cancelled
  `ctx` they stop waiting. That is why passing `ctx` matters.

## Settings and what they mean

| Config field | pgxpool field | Meaning |
|--------------|---------------|---------|
| `MaxOpenConns` | `MaxConns` | Upper limit of connections |
| `MaxIdleConns` | `MinConns` (closest equivalent) | Keep this many warm |
| `ConnMaxLifetime` (s) | `MaxConnLifetime` | Recycle connections after this age (helps with DB failover and load-balancer rebalancing) |
| `ConnMaxIdleTime` (s) | `MaxConnIdleTime` | Close connections idle this long |

> In the current code these are read from config but **not applied**. See
> [`internal/database/README.md`](../internal/database/README.md) for the
> 4-line fix.

## Query patterns with pgx v5

```go
// one row → struct
rows, _ := pool.Query(ctx, `SELECT * FROM todos WHERE id = $1`, id)
todo, err := pgx.CollectOneRow(rows, pgx.RowToAddrOfStructByName[model.Todo])

// many rows → slice
rows, _ := pool.Query(ctx, `SELECT * FROM todos WHERE user_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3`, uid, limit, offset)
todos, err := pgx.CollectRows(rows, pgx.RowToStructByName[model.Todo])

// exec (no rows)
tag, err := pool.Exec(ctx, `DELETE FROM todos WHERE id = $1 AND user_id = $2`, id, uid)
if tag.RowsAffected() == 0 { /* not found or not yours */ }

// named args
pool.Query(ctx, `INSERT INTO todos (title, user_id) VALUES (@title, @uid) RETURNING *`,
    pgx.NamedArgs{"title": t, "uid": uid})
```

## Transactions

```go
err := pgx.BeginFunc(ctx, pool, func(tx pgx.Tx) error {
    if _, err := tx.Exec(ctx, `UPDATE accounts SET balance = balance - $1 WHERE id = $2`, amt, from); err != nil {
        return err
    }
    if _, err := tx.Exec(ctx, `UPDATE accounts SET balance = balance + $1 WHERE id = $2`, amt, to); err != nil {
        return err
    }
    return nil   // commit; returning an error → rollback
})
```

## Performance checklist

- Index every foreign key and every `WHERE` / `ORDER BY` column you use often.
- `SELECT` only the columns you need on hot paths.
- Paginate with `LIMIT` (and keyset pagination for very large tables).
- Watch out for **N+1 queries**: one query per item in a loop. Use `JOIN` or
  `WHERE id = ANY($1)`.
- Use `EXPLAIN ANALYZE` on slow queries; New Relic shows which ones are slow.

---

Next: [13 · Migrations workflow](13-migrations-workflow.md)
