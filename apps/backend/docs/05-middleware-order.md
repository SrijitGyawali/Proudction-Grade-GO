# 05 · Middleware order: the onion

`router.Use(a, b, c)` means **a wraps b wraps c wraps the handler**. On the way
in, `a` runs first; on the way out, `a` finishes last.

```
          ┌───────────── RateLimiter ─────────────┐
          │ ┌─────────── CORS ───────────────────┐ │
          │ │ ┌───────── Secure ───────────────┐ │ │
          │ │ │ ┌─────── RequestID ──────────┐ │ │ │
          │ │ │ │ ┌───── NewRelic ─────────┐ │ │ │ │
          │ │ │ │ │ ┌─── EnhanceTracing ─┐ │ │ │ │ │
          │ │ │ │ │ │ ┌─ EnhanceContext ┐ │ │ │ │ │ │
          │ │ │ │ │ │ │ ┌ RequestLogger ┐│ │ │ │ │ │ │
          │ │ │ │ │ │ │ │ ┌ Recover ──┐ ││ │ │ │ │ │ │
 request ─┼─┼─┼─┼─┼─┼─┼─┼─┼─► handler │ ││ │ │ │ │ │ │
          │ │ │ │ │ │ │ │ └───────────┘ ││ │ │ │ │ │ │
          ...                                          ...
```

## Rules for choosing an order

1. **Cheap rejections first.** Rate limiting and CORS pre-flight should stop bad
   or pointless requests before any expensive work.
2. **Producers before consumers.** RequestID must run before anything that
   *reads* the request id (tracing, logging). NewRelic must run before
   EnhanceTracing, which reads the transaction.
3. **Loggers wrap the handler closely** so they see the final status and
   latency.
4. **Recover is innermost** so a panic in the handler becomes a normal `error`
   that outer middleware (logger, tracing, error handler) can process.
5. **Auth is per-route**, not global, because `/status` and `/docs` must stay
   public.

## Experiment: what breaks if you reorder?

| Change | Consequence |
|--------|-------------|
| Put `RequestID` after `EnhanceContext` | Request logger has an empty `request_id` |
| Put `Recover` first (outermost) | A panic skips RequestLogger and tracing, so you lose the log line for the crashing request |
| Put `RateLimiter` last | Abusive traffic still pays for tracing, logging, CORS… |
| Make `RequireAuth` global | Health checks get 401, and the load balancer marks every instance as dead |

## Where the error handler fits

`router.HTTPErrorHandler = GlobalErrorHandler` is **not** a middleware. Echo
calls it once, at the very end, with whatever error bubbled out of the whole
chain.

```
handler returns err
 → Recover passes it through
 → RequestLogger reads it (to pick status for the log)
 → ... outer middleware ...
 → Echo calls GlobalErrorHandler(err, c) → writes JSON
```

That is why `RequestLogger` contains the comment *"status code is not set yet as
it gets picked up by the global err handler"*. It reads the status from the
error itself.

## Writing your own middleware

```go
// internal/middleware/timeout.go
func Timeout(d time.Duration) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            ctx, cancel := context.WithTimeout(c.Request().Context(), d)
            defer cancel()
            c.SetRequest(c.Request().WithContext(ctx))
            return next(c)
        }
    }
}
```

Then add it in `router.go` at the right position (just before `Recover` is a
good spot).

---

Next: [06 · Dependency injection](06-dependency-injection.md)
