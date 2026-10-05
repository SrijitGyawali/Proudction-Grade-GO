# `internal/sqlerr/` — turning Postgres errors into friendly API errors

## The problem

When an insert breaks a rule, Postgres returns something like:

```
ERROR: duplicate key value violates unique constraint "users_email_key" (SQLSTATE 23505)
```

You must **not** send that to a client: it leaks your schema and means nothing
to users. You *do* want to send a clear `400` with a stable code:

```json
{ "code": "USER_ALREADY_EXISTS", "message": "A User with this Email already exists", "status": 400, "override": true }
```

This package does that translation **automatically for every endpoint**.
Repositories can just return the raw error.

## Files

| File         | What it contains |
|--------------|------------------|
| `error.go`   | Our own `Error` type plus enums: `Code` (`UniqueViolation`, `ForeignKeyViolation`, `NotNullViolation`, `CheckViolation`, `DeadlockDetected`, …) mapped from **SQLSTATE** codes by `MapCode`, and `Severity` mapped by `MapSeverity`. `Unwrap()` keeps the original driver error reachable. |
| `handler.go` | `HandleError(err)`, the converter called by `middleware.GlobalErrorHandler`, plus helpers: `ConvertPgError`, `generateErrorCode`, `formatUserFriendlyMessage`, `getEntityName`, `humanizeText`, `extractColumnForUniqueViolation`, `ErrCode` |

## SQLSTATE → behaviour

| SQLSTATE | Meaning                | HTTP | Code example           | Message example |
|----------|------------------------|------|------------------------|-----------------|
| `23505`  | unique violation       | 400  | `USER_ALREADY_EXISTS`  | "A User with this Email already exists" |
| `23503`  | foreign key violation  | 400  | `TODO_NOT_FOUND`       | "The referenced Project does not exist" |
| `23502`  | not null violation     | 400  | `TODO_REQUIRED`        | "The Title is required" (+ field error) |
| `23514`  | check violation        | 400  | `TODO_INVALID`         | "The Priority value does not meet required conditions" |
| other    | anything else          | 500  | `INTERNAL_SERVER_ERROR`| generic |
| —        | `pgx.ErrNoRows`        | 404  | `NOT_FOUND`            | "Resource not found" / "Todo not found" |

## How names are derived

- **Code**: `UPPER(table name)`, singularised by dropping a trailing `S`, +
  `_ALREADY_EXISTS` / `_NOT_FOUND` / `_REQUIRED` / `_INVALID`.
  `users` → `USER_ALREADY_EXISTS`.
- **Entity name**: from a `xxx_id` column if present (`project_id` →
  "Project"), else from the table name.
- **Unique field**: parsed from the **constraint name**: `unique_users_email`
  or `users_email_key` → "Email".
- **Not-found table**: if the error text contains `table:todos:`, the message
  becomes "Todo not found". So in a repository:
  `return nil, fmt.Errorf("table:todos: %w", pgx.ErrNoRows)`.

**Takeaway:** name your constraints carefully in migrations. Good names give
good error messages for free.

## Limitations

- Singularising by dropping "s" is naive: `categories` → `CATEGORIE`,
  `addresses` → `ADDRESSE`. Fine for most tables; override with an explicit
  `errs.*` error in the service when it matters.
- Deadlocks and too-many-connections become plain 500s. You might want to retry
  deadlocks, or return 503 for connection exhaustion.
