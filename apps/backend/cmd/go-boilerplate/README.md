# `cmd/go-boilerplate/` — the API server entry point

This folder contains exactly one file: `main.go`. It is the first code that runs
when you start the backend.

## What `main.go` does, step by step

```
1. config.LoadConfig()                  read BOILERPLATE_* env vars into a struct
2. logger.NewLoggerService(...)         start New Relic (if a license key is set)
3. logger.NewLoggerWithService(...)     build the zerolog logger
4. database.Migrate(...)                run SQL migrations (only when env != "local")
5. server.New(...)                      connect Postgres + Redis, start the job worker
6. repository.NewRepositories(srv)      data-access layer
7. service.NewServices(srv, repos)      business-logic layer
8. handler.NewHandlers(srv, services)   HTTP layer
9. router.NewRouter(srv, handlers, ...) Echo router + middleware + routes
10. srv.SetupHTTPServer(r)              wrap router in net/http.Server with timeouts
11. go srv.Start()                      listen on the port (in a goroutine)
12. <-ctx.Done()                        block until Ctrl+C (SIGINT)
13. srv.Shutdown(ctx)                   graceful shutdown with a 30 s deadline
```

## Things to notice

### 1. The order of construction follows the dependency graph

`repositories` need the server (for the DB pool). `services` need
repositories. `handlers` need services. `router` needs handlers. Each layer
gets **only what it needs** through its constructor. This is
*constructor-based dependency injection*. There is no DI framework and no magic.

```
config → logger → server(DB, Redis, Jobs) → repos → services → handlers → router → http.Server
```

### 2. The server runs in a goroutine

`srv.Start()` calls `ListenAndServe()`, which **blocks forever**. If we called it
directly, `main` could never reach the shutdown code. So it runs in a goroutine,
and the main goroutine waits on `ctx.Done()` instead.

`http.ErrServerClosed` is **expected** during shutdown, so it is not treated as
a failure.

### 3. Graceful shutdown

`signal.NotifyContext(..., os.Interrupt)` gives us a context that is cancelled
when you press Ctrl+C. Then:

- `httpServer.Shutdown(ctx)` stops accepting new connections and waits for
  in-flight requests to finish (up to `DefaultContextTimeout` = 30 s)
- the DB pool is closed
- the Asynq job server is stopped

Without this, a deploy would cut requests off in the middle.

### 4. Migrations are skipped locally

```go
if cfg.Primary.Env != "local" { database.Migrate(...) }
```

In staging/production the app migrates itself on boot. Locally you run
`task migrations:up` yourself, so you stay in control while you develop.

### 5. `log.Fatal` vs `defer`

`log.Fatal()` calls `os.Exit(1)`, and **deferred functions do not run** after
`os.Exit`. So `defer loggerService.Shutdown()` is skipped on a fatal error.
That is acceptable here because a fatal error at boot means nothing useful has
happened yet. It is still worth knowing.

### 6. Only `os.Interrupt` is handled

Docker and Kubernetes stop containers with **SIGTERM**, not SIGINT. For
production you would usually write:

```go
signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
```

See [`../../docs/19-graceful-shutdown.md`](../../docs/19-graceful-shutdown.md).
