# `internal/server/` — the application container

## Job of this package

Hold **every long-lived shared resource** in one struct and manage its
**lifecycle** (create → start → shutdown).

```go
type Server struct {
    Config        *config.Config            // all settings
    Logger        *zerolog.Logger           // base logger
    LoggerService *logger.LoggerService     // New Relic app (may be nil inside)
    DB            *database.Database        // Postgres pool
    Redis         *redis.Client             // Redis client
    Job           *job.JobService           // Asynq client + worker
    httpServer    *http.Server              // private: only Start/Shutdown touch it
}
```

Repositories, services, handlers and middleware all receive `*server.Server`, so
they can reach `s.DB.Pool`, `s.Config`, `s.Job.Client`, and so on.

## Files

| File        | What it contains |
|-------------|------------------|
| `server.go` | `Server` struct, `New()` (connect DB, connect Redis, start job worker), `SetupHTTPServer(handler)`, `Start()`, `Shutdown(ctx)` |

## `New()`: what happens at boot

```
database.New()            → must succeed, otherwise return error (app won't start)
redis.NewClient()         → + New Relic hook if enabled
redis Ping (5 s timeout)  → on failure: log error and CONTINUE (Redis is optional at boot)
job.NewJobService()       → Asynq client + server (concurrency 10, 3 queues)
jobService.InitHandlers() → build email client for job handlers
jobService.Start()        → begin consuming tasks in background goroutines
```

Note the deliberate difference: **Postgres is required, Redis is not.** The API
can still serve reads if Redis is down; only jobs and caching suffer.

## `SetupHTTPServer`: why wrap Echo in `http.Server`?

Echo could listen by itself, but a plain `net/http.Server` lets us set
**timeouts**, which are essential in production:

| Timeout        | Protects against |
|----------------|------------------|
| `ReadTimeout`  | Slow clients sending the request very slowly (Slowloris attack) |
| `WriteTimeout` | Handlers or clients that never finish the response |
| `IdleTimeout`  | Keep-alive connections that hold resources forever |

Go's default for all of these is **no timeout at all**.

## `Shutdown(ctx)`: order matters

```
1. httpServer.Shutdown(ctx)  stop accepting, finish in-flight requests
2. DB.Close()                now nothing can be using the pool
3. Job.Stop()                stop worker, close Asynq client
```

Close things in **reverse order of dependency**: first stop what *produces*
work (HTTP), then what it *uses* (DB, jobs).

Note: `Redis` is not closed explicitly here. Adding `s.Redis.Close()` would be
tidy.

## Is this a "god object"?

Passing one big `*Server` everywhere is **pragmatic**. It keeps constructors
short, and adding a dependency means adding one field. The trade-off is that
every package can see everything. A stricter style passes only what each piece
needs (`NewTodoRepository(pool *pgxpool.Pool)`). Both are common in production
Go. Be consistent within a project.
