# 10 · Logging in production

## What a good log line looks like

```json
{
  "level": "error",
  "time": "2026-10-05 14:03:11",
  "service": "boilerplate",
  "environment": "production",
  "request_id": "4c90fc3f-39cc-4b04-af21-c83ee64aa67e",
  "trace.id": "a1b2c3...",
  "method": "POST",
  "path": "/api/v1/todos",
  "ip": "203.0.113.7",
  "user_id": "user_2abc",
  "status": 400,
  "error_code": "TODO_ALREADY_EXISTS",
  "error": "insert todo: ERROR: duplicate key value ... (SQLSTATE 23505)",
  "message": "A Todo with this Title already exists"
}
```

Every field answers a question you'll ask during an incident: *who? which
request? which endpoint? what failed? how do I find the trace?*

## Three loggers in this codebase

| Logger | Built in | Has | Use it for |
|--------|----------|-----|------------|
| **Base** (`server.Logger`) | `logger.NewLoggerWithService` | service, environment | startup/shutdown, background goroutines |
| **Request** (`middleware.GetLogger(c)`) | `middleware.ContextEnhancer` | + request_id, method, path, ip, trace ids | **anything inside a request** |
| **SQL** (pgx) | `logger.NewPgxLogger` | component=database | automatic, local env only |

## Levels: when to use which

| Level | Use for | Example |
|-------|---------|---------|
| `debug` | Detailed internals, noisy | "validation successful", SQL params |
| `info` | Normal, notable events | "server started", "email sent", the per-request "API" line |
| `warn` | Something odd but handled; client errors (4xx) | "rate limit exceeded", 404s |
| `error` | Something failed and someone may need to look | 5xx, failed job, failed health check |
| `fatal` | Can't continue → process exits | bad config at startup |

Production usually runs at `info`. Turn on `debug` temporarily when hunting a bug.

## zerolog cheat-sheet

```go
log := middleware.GetLogger(c)

log.Info().Str("todo_id", id.String()).Msg("todo created")
log.Warn().Int("count", n).Msg("user near todo limit")
log.Error().Err(err).Str("to", email).Msg("failed to send email")
log.Debug().Interface("req", req).Msg("incoming payload")   // careful with secrets/PII

// child logger with extra fields for a whole function
l := log.With().Str("operation", "import_csv").Logger()
l.Info().Int("rows", 120).Msg("parsed")
```

**Always end with `.Msg()` or `.Send()`**, otherwise nothing is written.

## Console vs JSON

- **Local/dev**: `ConsoleWriter`, coloured and human-friendly.
- **Production + `LOGGING.FORMAT=json`**: one JSON object per line on stdout. The
  platform (Docker, k8s, New Relic) collects stdout. **Apps should never manage
  log files themselves.**

## Golden rules

1. **Log at the boundary, once.** The global error handler logs failed requests,
   so don't also log the same error in every layer.
2. **Use fields, not string formatting.** `Str("user_id", id)` is searchable;
   `Msgf("user %s", id)` is not.
3. **Never log secrets or personal data**: passwords, tokens, API keys, full
   card numbers, full request bodies.
4. **Include the request id** by using the request logger, which happens
   automatically when you call `GetLogger(c)`.
5. **Messages are constant strings.** Put the variable parts in fields, so you
   can count occurrences of "failed to send email".

---

Next: [11 · Tracing & New Relic](11-tracing-new-relic.md)
