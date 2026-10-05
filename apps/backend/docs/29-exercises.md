# 29 · Exercises: learn by changing the code

Do these in order. Each one teaches a specific part of the architecture. Make a
branch and a commit per exercise.

## Level 1: Warm-up (no new layers)

1. **Run it.** Start Postgres + Redis (Docker), fill `.env`, `task run`, open
   `/status` and `/docs`. Stop Redis and call `/status` again. What changes?
2. **Add `/livez`.** A route in `router/system.go` that returns `204`. Why
   shouldn't it check the DB?
3. **Handle SIGTERM** in `main.go`. Test with `kill -TERM <pid>` (Linux/macOS)
   or `docker stop`.
4. **Close Redis** in `Server.Shutdown`.

## Level 2: Infrastructure fixes

5. **Apply pool settings** from config to `pgxpool` in `database.New`. Log
   the final pool config at startup.
6. **Support multiple CORS origins.** Use `k.UnmarshalWithConf` with a
   `StringToSliceHookFunc(",")` decode hook.
7. **Move `RequestID()` before the rate limiter** and confirm the rate-limit log
   now has a `request_id`.
8. **Add a body-size limit** (`echoMiddleware.BodyLimit("1M")`) and test it with
   a 2 MB payload.
9. **Add a request-timeout middleware** (context with deadline). Write a handler
   that sleeps 10 s and confirm it is cut off.

## Level 3: Your first feature

10. **Todos**: follow [21 · Adding a feature](21-adding-a-feature.md) exactly.
11. **Ownership**: user A must not delete user B's todo. Write a test.
12. **Update endpoint**: `PATCH /api/v1/todos/:id` with optional fields
    (`*string`, `*bool`). Keep `updated_at` correct.
13. **Pagination**: add `?completed=true` filtering and make sure an index
    supports it.

## Level 4: Async & integrations

14. **Enqueue the welcome email** when a user is created (from a Clerk webhook
    or a fake `POST /api/v1/users`).
15. **Make the email job idempotent** using `asynq.Unique` or a `sent_emails`
    table.
16. **Add a second email template** (password changed) end to end, from the
    React Email `.tsx` file to `SendPasswordChangedEmail`.
17. **Split the worker** into `cmd/worker/main.go` so the API only enqueues.

## Level 5: Quality

18. **Fix the shared-request-pointer race** in `handler.Handle` (use a factory)
    and prove it with `go test -race` and concurrent requests.
19. **Make `extractValidationErrors` panic-free** with `errors.As`.
20. **Unit-test `sqlerr.HandleError`** with fake `*pgconn.PgError` values for
    each SQLSTATE.
21. **Run `golangci-lint run ./...`**, then fix every issue in one package.
22. **Write a Dockerfile** (multi-stage, distroless, non-root) and a
    `docker-compose.yml` with Postgres + Redis + API.
23. **Add a GitHub Actions workflow**: lint, test with `-race`, govulncheck,
    build.

## Stretch goals

24. Replace the in-memory rate limiter with a Redis-backed one.
25. Add OpenTelemetry tracing alongside New Relic.
26. Add `sqlc` and generate one repository from SQL files.
27. Add an outbox table and poller for reliable job enqueueing.

When you can do all of Level 3 without looking at the guide, you understand
the architecture.
