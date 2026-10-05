# `internal/errs/` — the API's single error language

## Job of this package

Define **one error type**, `*errs.HTTPError`, that every layer can return and that
the global error handler knows how to turn into JSON.

This gives every client (web, mobile, other services) the **same error shape**
for every failure.

## Files

| File       | What it contains |
|------------|------------------|
| `http.go`  | The `HTTPError` struct, `FieldError`, `Action`/`ActionType`, the `Error()` / `Is()` / `WithMessage()` methods, and the `MakeUpperCaseWithUnderscores` helper |
| `types.go` | Constructors: `NewBadRequestError`, `NewUnauthorizedError`, `NewForbiddenError`, `NewNotFoundError`, `NewInternalServerError`, `ValidationError` |

## The JSON shape the client receives

```json
{
  "code": "BAD_REQUEST",
  "message": "Validation failed",
  "status": 400,
  "override": true,
  "errors": [
    { "field": "email", "error": "must be a valid email address" }
  ],
  "action": null
}
```

| Field      | Meaning |
|------------|---------|
| `code`     | Machine-readable, stable string. Frontends switch on this (`USER_ALREADY_EXISTS`), never on `message`. |
| `message`  | Human-readable text. |
| `status`   | The HTTP status code, repeated in the body for convenience. |
| `override` | `true` means the message is safe and specific enough to show to the end user as-is. `false` means the frontend should show its own generic text. |
| `errors`   | Per-field problems, used to highlight form inputs. |
| `action`   | Optional instruction for the client, e.g. `{ "type": "redirect", "value": "/login" }`. |

## Why implement `error`?

```go
func (e *HTTPError) Error() string { return e.Message }
```

Because `*HTTPError` satisfies Go's built-in `error` interface, a service can just
`return errs.NewNotFoundError(...)` from a function whose signature is
`(..., error)`. No special plumbing is needed. Later, the global handler uses
`errors.As(err, &httpErr)` to get the details back.

## How to use it

```go
// in a service
if todo == nil {
    code := "TODO_NOT_FOUND"
    return nil, errs.NewNotFoundError("Todo not found", true, &code)
}

// forbidden
if todo.OwnerID != userID {
    return nil, errs.NewForbiddenError("You cannot edit this todo", true)
}

// anything unexpected: just return the raw error.
// The global handler turns it into a 500 and logs the real cause.
return nil, fmt.Errorf("loading todo %s: %w", id, err)
```

## Mental model

```
  any layer returns error
          │
          ▼
middleware.GlobalErrorHandler
   ├─ already *errs.HTTPError?   → use it
   ├─ *echo.HTTPError?           → convert (404 → "Route not found")
   ├─ Postgres error?            → sqlerr.HandleError → *errs.HTTPError
   └─ anything else              → 500 INTERNAL_SERVER_ERROR (details only in logs)
```

**Never leak internal error text (SQL, stack traces) to clients.** 500s always
say just "Internal Server Error". The real cause goes to the logs with the
request id.

See [`../../docs/09-error-handling.md`](../../docs/09-error-handling.md).
