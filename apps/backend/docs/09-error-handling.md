# 09 · Error handling strategy

## The goals

1. **Clients** get a **consistent, safe** JSON error with a stable `code`.
2. **Developers** get the **full original error**, stack trace and request id in logs.
3. **Code** in handlers/services stays simple: just `return err`.

## Three kinds of errors

| Kind | Example | Who creates it | Client sees |
|------|---------|----------------|-------------|
| **Expected / domain** | not found, forbidden, limit reached, invalid input | service / validation via `errs.New…` | specific 4xx + `code` + message |
| **Database constraint** | duplicate email, missing FK | Postgres → translated by `sqlerr` | friendly 400 |
| **Unexpected** | network blip, nil pointer, bug | anywhere | generic **500**, no details |

## The flow

```
           return err   return err   return err
repository ─────────► service ─────► handler ─────► Echo
                                                      │
                                       GlobalErrorHandler(err, c)
                                          │
              ┌───────────────────────────┼────────────────────────────┐
              ▼                           ▼                            ▼
       *errs.HTTPError              *echo.HTTPError               anything else
       (use as-is)              (404 → "Route not found",      sqlerr.HandleError(err)
                                 else map status)                ├─ PgError → 400 friendly
                                                                 ├─ ErrNoRows → 404
                                                                 └─ other → 500
              └───────────────────────────┬────────────────────────────┘
                                          ▼
               log ORIGINAL error (stack, request_id, user_id, trace)
                                          ▼
                    c.JSON(status, errs.HTTPError{...})
```

## Writing errors well in Go

### Wrap with context using `%w`

```go
if err != nil {
    return nil, fmt.Errorf("todo service: create for user %s: %w", userID, err)
}
```

- `%w` keeps the original error **inside**, so `errors.As` / `errors.Is` still
  find the `*pgconn.PgError` or `*errs.HTTPError` underneath.
- The message builds a readable chain:
  `todo service: create for user u1: insert todo: ERROR: duplicate key…`

### Use `errors.As` / `errors.Is`, never `==` on wrapped errors

```go
var httpErr *errs.HTTPError
if errors.As(err, &httpErr) { ... }      // type match anywhere in the chain
if errors.Is(err, pgx.ErrNoRows) { ... } // value match anywhere in the chain
```

### Handle each error once

Either **handle** it (log + respond) or **return** it, not both. Logging at
every layer produces 5 log lines for one failure. Here the global handler does
the final log, and `handleRequest` adds one line with timing.

### Don't panic for normal errors

`panic` is for "impossible" programmer bugs. `Recover` middleware catches
panics so the server survives, but it is a safety net, not a control-flow tool.

## Choosing `override`

`override: true` tells the frontend "this message is user-safe; show it". Use it
for messages you wrote for humans ("A User with this Email already exists").
Leave it `false` for technical or generic messages, and the frontend shows its
own copy.

## Stable error codes are part of your API

Frontends write `if (err.code === "TODO_LIMIT_REACHED") showUpgradeModal()`.
Changing a code is a **breaking change**, just like renaming a JSON field.

---

Next: [10 · Logging](10-logging.md)
