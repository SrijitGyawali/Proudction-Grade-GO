# 25 · Gotchas & known rough edges

These are real behaviours of the current code, found while reading it. They are
not criticism: every boilerplate has them. Knowing them saves debugging time,
and each one is a good **first contribution**.

## Configuration

| # | Gotcha | Effect | Fix |
|---|--------|--------|-----|
| 1 | `.env.sample` has `BOILERPLATE_REDIS.ADDRESS="redis://localhost:6379"` | go-redis and Asynq expect `host:port`. Redis ping fails, jobs never process, health shows Redis unhealthy | Use `localhost:6379` |
| 2 | List values aren't split on commas (koanf default decoder) | `CORS_ALLOWED_ORIGINS="a,b"` becomes one origin `"a,b"` | Use one origin, or add `StringToSliceHookFunc(",")` via `UnmarshalWithConf` |
| 3 | Env var names contain dots | Can't `export` them in bash | Use `.env`, `env "K.V=x" cmd`, or platform settings |
| 4 | Setting any `OBSERVABILITY.*` var disables all defaults for that section | Missing fields fail validation | Set the whole section or none |
| 5 | `OBSERVABILITY.ENVIRONMENT` / `SERVICE_NAME` are overwritten in code | Setting them has no effect | Set `PRIMARY.ENV` instead |

## Database

| # | Gotcha | Fix |
|---|--------|-----|
| 6 | Pool size settings (`MAX_OPEN_CONNS`, …) are not applied to pgxpool | See [`internal/database/README.md`](../internal/database/README.md) |
| 7 | `local` env never auto-migrates | Run `task migrations:up` (needs `BOILERPLATE_DB_DSN`) |
| 8 | Backend README mentions `task migrations:down`, which does not exist in the Taskfile | `tern migrate --destination -1`, or add the task |

## HTTP layer

| # | Gotcha | Effect | Fix |
|---|--------|--------|-----|
| 9 | `handler.Handle(..., req)` reuses **the same `req` pointer** for every request | Concurrent requests can bind into the same struct (data race) | Pass a factory `func() Req` and create a fresh value per request |
| 10 | `router.Group("/api/v1")` return value is discarded | Placeholder only | `v1 := router.Group(...)` and register routes on it |
| 11 | `EnhanceContext` runs before route-level `RequireAuth` | Request logger lacks `user_id` | Rebuild the logger in `RequireAuth` |
| 12 | Rate limiter runs before `RequestID` | Rate-limit log has empty `request_id` | Move `RequestID()` first |
| 13 | In-memory rate limiter | Limit is per replica | Redis-backed store |
| 14 | `validation.extractValidationErrors` asserts `err.(CustomValidationErrors)` without `ok` | Any other error type from `Validate()` → **panic** (→ 500 via Recover) | Use `errors.As` with a fallback |
| 15 | Bind errors are parsed with string splitting | Unexpected format → panic | Use `errors.As(err, &echo.HTTPError)` |
| 16 | `validator.New()` per call in typical `Validate()` impls | Loses the validator's cache | Use one package-level validator |

## Lifecycle

| # | Gotcha | Fix |
|---|--------|-----|
| 17 | Only `os.Interrupt` handled | Add `syscall.SIGTERM` for Docker/Kubernetes |
| 18 | Redis client not closed on shutdown | `s.Redis.Close()` in `Server.Shutdown` |
| 19 | `log.Fatal` skips `defer`s | Acceptable at boot; be aware |

## Assets & paths

| # | Gotcha | Fix |
|---|--------|-----|
| 20 | `static/` and `templates/` are read with **relative paths** | Run from `apps/backend`, copy folders in Docker, or `//go:embed` them |
| 21 | Email `From` is hard-coded to Resend's sandbox sender | Move to config, verify your domain |

## Misc

| # | Gotcha | Fix |
|---|--------|-----|
| 22 | `job.emailClient` is a package-level global | Make it a field of `JobService` |
| 23 | `sqlerr` singularises by trimming a trailing "s" | `categories` → `CATEGORIE`. Return explicit errors where it matters |
| 24 | `/docs` + `openapi.json` are public | Hide in production if needed |
| 25 | `internal/testing` shares its name with std-lib `testing` | Import with alias `testutil` |
