# 27 · File map: every backend file in one line

## Root of `apps/backend/`

| File | Purpose |
|------|---------|
| `go.mod` / `go.sum` | Module path + pinned dependencies with checksums |
| `Taskfile.yml` | Developer commands: run, migrations, tidy |
| `.env.sample` | Template of every env var (copy to `.env`) |
| `.golangci.yml` | Linter configuration |
| `.gitignore` | Keeps binaries and `.env` out of git |
| `README.md` | Original overview of the backend |

## `cmd/`

| File | Purpose |
|------|---------|
| `cmd/go-boilerplate/main.go` | Composition root: load config, build everything, serve, shut down gracefully |

## `internal/config/`

| File | Purpose |
|------|---------|
| `config.go` | `Config` structs + `LoadConfig()` (env → koanf → struct → validate) |
| `observability.go` | Logging/New Relic/health-check config, defaults, validation, `GetLogLevel`, `IsProduction` |

## `internal/database/`

| File | Purpose |
|------|---------|
| `database.go` | Build DSN, configure tracers (`multiTracer`), create + ping `pgxpool` |
| `migrator.go` | Embed `migrations/*.sql` and apply them with tern |
| `migrations/001_setup.sql` | Empty example migration showing the up/down format |

## `internal/errs/`

| File | Purpose |
|------|---------|
| `http.go` | `HTTPError`, `FieldError`, `Action` types + helpers |
| `types.go` | Constructors for 400/401/403/404/500 errors |

## `internal/handler/`

| File | Purpose |
|------|---------|
| `base.go` | Generic `Handle`/`HandleNoContent`/`HandleFile` with validation, logging, tracing |
| `handlers.go` | `Handlers` aggregate + constructor |
| `health.go` | `GET /status`: DB + Redis checks |
| `openapi.go` | `GET /docs`: serves Scalar UI HTML |

## `internal/lib/`

| File | Purpose |
|------|---------|
| `email/client.go` | Resend client + generic `SendEmail` (template render + send) |
| `email/emails.go` | `SendWelcomeEmail` |
| `email/templates.go` | Template name constants |
| `email/preview.go` | Sample data for templates |
| `job/job.go` | `JobService`: Asynq client/server, queues, start/stop |
| `job/email_tasks.go` | Task type + payload + constructor for welcome email |
| `job/handlers.go` | Task handler that sends the welcome email |
| `utils/utils.go` | `PrintJSON` debug helper |

## `internal/logger/`

| File | Purpose |
|------|---------|
| `logger.go` | New Relic app, zerolog setup, trace-context logger, pgx SQL logger |

## `internal/middleware/`

| File | Purpose |
|------|---------|
| `middlewares.go` | `Middlewares` aggregate + constructor |
| `global.go` | CORS, request logger, recover, secure headers, **global error handler** |
| `request_id.go` | `X-Request-ID` generation/propagation |
| `tracing.go` | New Relic transaction + custom attributes |
| `context.go` | Per-request logger; `GetLogger`, `GetUserID` |
| `auth.go` | Clerk `RequireAuth` |
| `rate_limit.go` | New Relic event on rate-limit hits |

## `internal/model/`

| File | Purpose |
|------|---------|
| `base.go` | `Base` (id/created/updated) + generic `PaginatedResponse[T]` |

## `internal/repository/`

| File | Purpose |
|------|---------|
| `repositories.go` | Empty `Repositories` aggregate, ready for your first repo |

## `internal/router/`

| File | Purpose |
|------|---------|
| `router.go` | Echo setup, rate limiter, middleware order, route groups |
| `system.go` | `/status`, `/docs`, `/static` |

## `internal/server/`

| File | Purpose |
|------|---------|
| `server.go` | `Server` container: DB, Redis, jobs, HTTP server lifecycle |

## `internal/service/`

| File | Purpose |
|------|---------|
| `services.go` | `Services` aggregate + constructor |
| `auth.go` | Sets the Clerk secret key at startup |

## `internal/sqlerr/`

| File | Purpose |
|------|---------|
| `error.go` | SQLSTATE → `Code`, severity mapping, `Error` type |
| `handler.go` | `HandleError`: Postgres error → friendly `HTTPError` |

## `internal/testing/`

| File | Purpose |
|------|---------|
| `container.go` | Start Postgres in Docker, migrate, return pool |
| `server.go` | Minimal `Server` for tests |
| `helpers.go` | `SetupTest`, `MustMarshalJSON`, `ProjectRoot`, `Ptr` |
| `transaction.go` | Run test code inside commit/rollback transactions |
| `assertions.go` | UUID, timestamp and struct-equality assertions |

## `internal/validation/`

| File | Purpose |
|------|---------|
| `utils.go` | `Validatable`, `BindAndValidate`, error message mapping, `IsValidUUID` |

## Assets

| File | Purpose |
|------|---------|
| `static/openapi.json` | Generated OpenAPI spec |
| `static/openapi.html` | Scalar API reference page |
| `templates/emails/welcome.html` | Generated welcome email HTML |
