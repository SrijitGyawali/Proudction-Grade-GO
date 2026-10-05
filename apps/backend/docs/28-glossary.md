# 28 · Glossary

| Term | Meaning |
|------|---------|
| **APM** | Application Performance Monitoring: tools like New Relic that measure latency, errors and throughput |
| **Asynq** | Go library for background jobs stored in Redis |
| **Backoff** | Waiting longer between each retry (e.g. 1 s, 2 s, 4 s…) |
| **Binding** | Copying HTTP input (JSON body, query, path) into a Go struct |
| **Clerk** | Hosted authentication provider |
| **Composition root** | The single place (here `main.go`) where all objects are created and wired |
| **Context (`context.Context`)** | Go value carrying cancellation, deadlines and request-scoped data |
| **CORS** | Browser rule controlling which websites may call your API |
| **DI (Dependency Injection)** | Passing dependencies into a component instead of creating them inside |
| **DSN** | Data Source Name: the DB connection string |
| **DTO** | Data Transfer Object: a struct shaped for API input/output |
| **Echo** | The Go HTTP framework used here |
| **Embedding** | Go's way to include one struct's fields/methods in another |
| **Expand/contract** | Zero-downtime migration technique: add the new thing, migrate, remove the old |
| **Graceful shutdown** | Stop accepting work, finish current work, then exit |
| **Handler** | Function that turns an HTTP request into a response |
| **Idempotent** | Running it twice has the same effect as once |
| **`internal/`** | Go folder whose packages can't be imported by other modules |
| **JWT** | JSON Web Token: a signed token proving identity |
| **koanf** | Config library that loads env vars into structs |
| **Liveness / readiness** | "Is the process alive?" vs "can it serve traffic now?" |
| **Middleware** | Code that wraps handlers to add cross-cutting behaviour |
| **Migration** | Versioned SQL file that changes the DB schema |
| **Monorepo** | One git repo containing several apps/packages |
| **N+1 query** | Running one query per item in a loop instead of one query for all |
| **New Relic** | Observability vendor (APM, logs, traces) |
| **OpenAPI** | Standard JSON/YAML format describing an HTTP API |
| **pgx / pgxpool** | PostgreSQL driver / connection pool for Go |
| **Rate limiting** | Capping how many requests a client may make per time window |
| **Repository** | Layer that contains all SQL for one domain |
| **Request ID** | Unique id per request, used to correlate logs |
| **Resend** | Email-sending API |
| **Service** | Layer containing business rules |
| **Span / segment** | One timed step inside a trace (e.g. one SQL query) |
| **SQLSTATE** | 5-character Postgres error code (e.g. `23505` = unique violation) |
| **Structured logging** | Logs as key/value data (JSON) instead of free text |
| **Struct tag** | Back-quoted metadata on struct fields (`json:"id"`) |
| **tern** | Postgres migration tool |
| **Testcontainers** | Library that starts real services (Postgres) in Docker for tests |
| **Trace** | Record of one request's path through the system, with timings |
| **Transaction (DB)** | Group of SQL statements that all succeed or all fail |
| **Transaction (APM)** | New Relic's record of one request or job |
| **Turborepo** | Build orchestrator for JS/TS monorepos |
| **12-Factor App** | Set of principles for cloud-ready services (config in env, logs to stdout…) |
| **zerolog** | Fast structured logger for Go |
| **Zod** | TypeScript schema/validation library |
