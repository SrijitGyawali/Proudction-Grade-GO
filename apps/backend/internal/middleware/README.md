# `internal/middleware/` — code that wraps every request

## What is middleware?

A middleware is a function that takes the **next handler** and returns a **new
handler** that does something *before* and/or *after* calling it:

```go
func MyMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // ── before ── (e.g. start timer, check token)
        err := next(c)
        // ── after ──  (e.g. log status + latency)
        return err
    }
}
```

Stacked together they form an **onion** around your handler. This is where
cross-cutting concerns go: things *every* endpoint needs, but no endpoint should
implement itself.

## Files

| File              | What it contains |
|-------------------|------------------|
| `middlewares.go`  | `Middlewares` struct grouping all middleware objects; `NewMiddlewares(server)` builds them (and fetches the New Relic app) |
| `global.go`       | `GlobalMiddlewares`: `CORS()`, `RequestLogger()`, `Recover()`, `Secure()` and the very important **`GlobalErrorHandler`** |
| `request_id.go`   | `RequestID()`: reads `X-Request-ID` or generates a UUID, stores it in context, echoes it in the response header; `GetRequestID(c)` |
| `tracing.go`      | `TracingMiddleware`: `NewRelicMiddleware()` (starts a New Relic transaction per request, no-op if disabled) and `EnhanceTracing()` (adds ip, user-agent, request id, user id, status code; reports errors) |
| `context.go`      | `ContextEnhancer.EnhanceContext()`: builds a **per-request logger** (request_id, method, path, ip, trace ids, user) and stores it; helpers `GetLogger(c)` and `GetUserID(c)` |
| `auth.go`         | `AuthMiddleware.RequireAuth`: verifies the **Clerk** session token from the `Authorization` header and puts `user_id`, `user_role`, `permissions` into the context |
| `rate_limit.go`   | `RateLimitMiddleware.RecordRateLimitHit`: sends a `RateLimitHit` event to New Relic (the limiter itself is configured in `router.go`) |

## Order matters! (set in `router/router.go`)

```
request ─►  1. RateLimiter      (20 req/s per IP, in memory)  → 429 if exceeded
            2. CORS             (browser cross-origin rules)
            3. Secure           (security headers: XSS, nosniff, frame-options…)
            4. RequestID        (X-Request-ID)
            5. NewRelic         (start transaction)
            6. EnhanceTracing   (add attributes to transaction)
            7. EnhanceContext   (build request logger)
            8. RequestLogger    (one "API" log line per request, after it finishes)
            9. Recover          (turn panics into 500s instead of crashing)
           [route-level] RequireAuth   (only on protected routes)
           ─► handler
```

Why this order?
- **Rate limit first**, so abusive traffic costs as little CPU as possible.
- **RequestID before** tracing/logging, so they can include it.
- **EnhanceContext before RequestLogger**, so the final log line uses the
  enriched logger.
- **Recover last (innermost)**: it catches panics from the handler and turns
  them into errors that the logger and error handler can see.

## `GlobalErrorHandler`: one place for all errors

Echo calls this whenever a handler (or middleware) returns an error.

```
err
 ├─ *errs.HTTPError   → use its status/code/message/field errors
 ├─ *echo.HTTPError   → 404 becomes "Route not found"; others mapped by status
 ├─ Postgres error    → sqlerr.HandleError() → friendly *errs.HTTPError
 └─ unknown           → 500 INTERNAL_SERVER_ERROR
then: log the ORIGINAL error with stack + request context
      write JSON (unless the response was already sent)
```

## `RequireAuth`: how to protect routes

```go
api := router.Group("/api/v1")
api.Use(middlewares.Auth.RequireAuth)     // every route below needs a valid Clerk token
api.GET("/me", ...)
```

Then inside a handler: `userID := middleware.GetUserID(c)`.

## Gotchas

- `EnhanceContext` runs globally **before** `RequireAuth` (which is per-route),
  so `user_id` is **not yet set** when the request logger is built. To get
  `user_id` on every log line, refresh the logger inside `RequireAuth`, or run
  `EnhanceContext` again after it.
- The rate limiter runs **before** `RequestID`, so the "rate limit exceeded" log
  has an empty `request_id`.
- The in-memory rate limiter is **per process**. With 3 replicas, a client
  effectively gets 3× the limit. Use a Redis-backed limiter for a shared limit.
