# 19 · Graceful shutdown

## Why it matters

Every deploy **stops old instances**. If a process dies instantly:
- requests in flight get a connection reset (users see errors),
- a DB transaction may be half done,
- a background job may be interrupted mid-way.

Graceful shutdown means: **stop taking new work, finish current work, release
resources, exit.**

## What this codebase does (`main.go` + `server.Shutdown`)

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
go func() { srv.Start() }()       // serve in background
<-ctx.Done()                       // block until Ctrl+C

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
srv.Shutdown(ctx)
```

```
signal received
  │
  ├─ httpServer.Shutdown(ctx)
  │     • close listeners (no new connections)
  │     • close idle keep-alive connections
  │     • wait for active requests to finish … up to 30 s
  ├─ DB.Close()            pool closed after HTTP is drained
  ├─ Job.Stop()            Asynq: stop pulling tasks, wait for active ones,
  │                        unfinished tasks go back to Redis
  └─ "server exited properly"
```

## Improvements for container platforms

### 1. Handle SIGTERM

Docker (`docker stop`) and Kubernetes send **SIGTERM** first, then **SIGKILL**
after a grace period (default 30 s in k8s, 10 s in Docker).

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
```

Without this, SIGTERM uses Go's default behaviour and exits immediately, so the
graceful path never runs in production.

### 2. Keep the timeout below the platform's grace period

Shutdown timeout (30 s) should be **less** than `terminationGracePeriodSeconds`
(k8s), otherwise SIGKILL arrives first. Set the grace period to ~40 s or lower
the app timeout to ~25 s.

### 3. Fail readiness first (zero-downtime deploys)

Load balancers take a few seconds to notice a pod is going away. A common
pattern is: on SIGTERM, start returning 503 from the readiness endpoint, wait
~5 s, *then* call `Shutdown`.

### 4. Close everything

`Redis` isn't closed in `Server.Shutdown` today; add `s.Redis.Close()`. Also
flush telemetry: `loggerService.Shutdown()` runs via `defer` in `main`.

## Mental checklist for any long-running Go program

- [ ] Every goroutine has a way to be told to stop (`ctx`)
- [ ] Every resource opened at start is closed at shutdown, in reverse order
- [ ] Shutdown has a deadline
- [ ] SIGINT **and** SIGTERM are handled

---

Next: [20 · Testing strategy](20-testing-strategy.md)
