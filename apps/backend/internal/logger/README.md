# `internal/logger/` — structured logging + New Relic

## Job of this package

1. Start the **New Relic** agent (APM, tracing, log forwarding), if a license key exists.
2. Build the app's **zerolog** logger with the right level, format and base fields.
3. Provide a separate, pretty **SQL logger** for pgx.

## Files

| File        | What it contains |
|-------------|------------------|
| `logger.go` | `LoggerService` (owns the `*newrelic.Application`), `NewLoggerService`, `Shutdown`, `GetApplication`, `NewLoggerWithService`, `WithTraceContext`, `NewPgxLogger`, `GetPgxTraceLogLevel` |

## Why structured logging?

```
❌ fmt.Println("user 42 failed to login from 1.2.3.4")
✅ {"level":"warn","user_id":"42","ip":"1.2.3.4","request_id":"…","message":"login failed"}
```

Structured (JSON) logs can be **searched and filtered** by field in tools like
New Relic, Datadog, Loki or CloudWatch: "show all errors for request_id X",
"count failed logins per IP". That is not possible with free text.

**zerolog** is chosen because it is **zero-allocation** and very fast. The API
reads like a chain:

```go
log.Info().Str("user_id", id).Dur("took", d).Msg("user created")
```

## `LoggerService`: the New Relic part

```go
NewLoggerService(cfg) →
   no license key?  → returns an empty service (New Relic disabled, everything still works)
   license key      → newrelic.NewApplication(AppName, License, LogForwarding, DistributedTracing)
```

Other packages call `loggerService.GetApplication()` and **always check for
`nil`**. That is how the app runs with or without New Relic.

## `NewLoggerWithService`: the logger itself

| Environment           | Output |
|-----------------------|--------|
| `production` + `json` | JSON to stdout, wrapped by New Relic's `zerologWriter` so logs are **forwarded and linked to traces** |
| anything else         | Colourful human-readable `ConsoleWriter` |

Every log line automatically gets: `timestamp`, `service`, `environment`.
In non-production, `.Stack()` is enabled so errors include stack traces.

Levels: `debug` < `info` < `warn` < `error`. Logs below the configured level are dropped.

## `WithTraceContext`

Adds `trace.id` and `span.id` from the New Relic transaction to a logger. In New
Relic you can then jump from a slow trace straight to its log lines.
`middleware/context.go` calls this for every request.

## `NewPgxLogger` + `GetPgxTraceLogLevel`

A separate console logger tagged `component=database`, used only in `local`
env. It shows every SQL query, truncating long statements to 200 chars and
pretty-printing JSON arguments. `GetPgxTraceLogLevel` maps zerolog levels to
pgx's numeric tracelog levels.

## Logger hierarchy in a request

```
base logger (service, environment)                       ← built here, stored in Server.Logger
  └─ + request_id, method, path, ip, trace.id, user_id   ← middleware.ContextEnhancer
       └─ + operation, route                             ← handler.handleRequest
```

**Inside request code, always use `middleware.GetLogger(c)`**, not
`server.Logger`, so every line carries the request id.

## What NOT to log

Passwords, tokens, full credit-card numbers, API keys, or full request bodies
containing personal data. Logs are often kept for months and read by many people.
