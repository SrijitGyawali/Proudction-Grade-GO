# Production-Grade Go Backend: Learning Guide

This folder explains **how and why** this backend is built the way it is. Read it
alongside the code. Every folder in the backend also has its own `README.md`
describing its files.

## Suggested reading order

### Part 1: The big picture
1. [Mental model](01-mental-model.md): how to *think* about a production Go service
2. [Architecture diagram](02-architecture-diagram.md): boxes and arrows
3. [Startup flow](03-startup-flow.md): what happens when you run `main.go`
4. [Request lifecycle](04-request-lifecycle.md): one HTTP request, start to finish
5. [Middleware order](05-middleware-order.md): the onion around every handler
6. [Dependency injection](06-dependency-injection.md): how pieces get wired together

### Part 2: Infrastructure
7. [Configuration](07-configuration.md)
8. [Environment variable reference](08-env-reference.md)
9. [Error handling](09-error-handling.md)
10. [Logging](10-logging.md)
11. [Tracing & New Relic](11-tracing-new-relic.md)
12. [Database & connection pooling](12-database-pooling.md)
13. [Migrations workflow](13-migrations-workflow.md)
14. [Background jobs](14-background-jobs.md)
15. [Email pipeline](15-email-pipeline.md)
16. [Authentication with Clerk](16-authentication-clerk.md)
17. [Security](17-security.md)
18. [Health checks](18-health-checks.md)
19. [Graceful shutdown](19-graceful-shutdown.md)
20. [Testing strategy](20-testing-strategy.md)

### Part 3: Building on it
21. [Adding a feature end-to-end](21-adding-a-feature.md): a full Todo example
22. [Go patterns used in this codebase](22-go-patterns-used.md)
23. [Tooling: Taskfile, golangci-lint, go.mod](23-tooling.md)
24. [How the monorepo fits together](24-monorepo.md)
25. [Gotchas & known rough edges](25-gotchas.md)
26. [Production checklist](26-production-checklist.md)

### Reference
27. [File map: every file in one line](27-file-map.md)
28. [Glossary](28-glossary.md)
29. [Exercises](29-exercises.md): practise by extending the code

## Folder READMEs

| Folder | README |
|--------|--------|
| `cmd/` | [cmd/README.md](../cmd/README.md) |
| `cmd/go-boilerplate/` | [README](../cmd/go-boilerplate/README.md) |
| `internal/` | [internal/README.md](../internal/README.md) |
| `internal/config/` | [README](../internal/config/README.md) |
| `internal/database/` | [README](../internal/database/README.md) |
| `internal/database/migrations/` | [README](../internal/database/migrations/README.md) |
| `internal/errs/` | [README](../internal/errs/README.md) |
| `internal/handler/` | [README](../internal/handler/README.md) |
| `internal/lib/` | [README](../internal/lib/README.md) |
| `internal/lib/email/` | [README](../internal/lib/email/README.md) |
| `internal/lib/job/` | [README](../internal/lib/job/README.md) |
| `internal/lib/utils/` | [README](../internal/lib/utils/README.md) |
| `internal/logger/` | [README](../internal/logger/README.md) |
| `internal/middleware/` | [README](../internal/middleware/README.md) |
| `internal/model/` | [README](../internal/model/README.md) |
| `internal/repository/` | [README](../internal/repository/README.md) |
| `internal/router/` | [README](../internal/router/README.md) |
| `internal/server/` | [README](../internal/server/README.md) |
| `internal/service/` | [README](../internal/service/README.md) |
| `internal/sqlerr/` | [README](../internal/sqlerr/README.md) |
| `internal/testing/` | [README](../internal/testing/README.md) |
| `internal/validation/` | [README](../internal/validation/README.md) |
| `static/` | [README](../static/README.md) |
| `templates/` | [README](../templates/README.md) |
| `templates/emails/` | [README](../templates/emails/README.md) |
