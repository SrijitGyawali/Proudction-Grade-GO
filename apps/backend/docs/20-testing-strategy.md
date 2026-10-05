# 20 · Testing strategy

## What to test at each layer

| Layer | Test style | Dependencies | Speed |
|-------|-----------|--------------|-------|
| `validation`, `sqlerr`, `errs`, utils | pure **unit tests**, table-driven | none | µs |
| `service` | unit with **fakes** (via small interfaces) **or** integration | fake repo / real DB | ms / s |
| `repository` | **integration** with real Postgres | Testcontainers | seconds |
| `handler` + router | **HTTP tests** with `httptest` | fake services or real stack | ms |
| full system | a few **end-to-end** smoke tests | everything | slow |

Aim for many fast tests at the bottom, and fewer slow ones at the top (the
testing pyramid).

## Table-driven tests: the idiomatic Go style

```go
func TestMapCode(t *testing.T) {
    tests := []struct {
        name string
        in   string
        want sqlerr.Code
    }{
        {"unique", "23505", sqlerr.UniqueViolation},
        {"fk", "23503", sqlerr.ForeignKeyViolation},
        {"unknown", "99999", sqlerr.Other},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            require.Equal(t, tt.want, sqlerr.MapCode(tt.in))
        })
    }
}
```

## Integration test with the toolkit

```go
func TestTodoRepo(t *testing.T) {
    testDB, srv, cleanup := testutil.SetupTest(t)   // Docker Postgres + migrations
    defer cleanup()
    _ = testDB

    repo := repository.NewTodoRepository(srv)
    todo, err := repo.Create(context.Background(), "u1", "x")
    require.NoError(t, err)
    testutil.AssertValidUUID(t, todo.ID)
}
```

Use `testutil.WithRollbackTransaction` to keep tests isolated without
re-creating the container.

## HTTP handler test

```go
func TestHealth(t *testing.T) {
    _, srv, cleanup := testutil.SetupTest(t)
    defer cleanup()

    h := handler.NewHealthHandler(srv)
    e := echo.New()
    req := httptest.NewRequest(http.MethodGet, "/status", nil)
    rec := httptest.NewRecorder()
    c := e.NewContext(req, rec)

    require.NoError(t, h.CheckHealth(c))
    require.Equal(t, http.StatusOK, rec.Code)
}
```

(`CreateTestServer` doesn't set Redis, so the health handler skips the Redis
check.)

## Commands

```bash
go test ./...                       # all
go test ./internal/sqlerr -run Map  # one package, filtered
go test -race ./...                 # detect data races (do this in CI!)
go test -cover ./...                # coverage summary
go test -short ./...                # convention: skip slow tests when -short
```

In slow integration tests:

```go
if testing.Short() { t.Skip("integration test") }
```

## What makes code testable

- Business logic in **services**, not handlers.
- Dependencies passed in (DI), not created inside.
- Small interfaces at the consumer side.
- `context.Context` passed everywhere (tests can cancel or timeout).
- No hidden globals (e.g. `job.emailClient` is a package global, so it is
  harder to swap in tests).

---

Next: [21 · Adding a feature](21-adding-a-feature.md)
