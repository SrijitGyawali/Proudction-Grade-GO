# `internal/testing/` — integration-test toolkit

## Job of this package

Make it easy to write tests that run against a **real PostgreSQL**, started
automatically in Docker, migrated, and thrown away after the test.

> Note: the package is named `testing`, the same as Go's standard library
> package. Import it with an alias to avoid confusion:
> `import testutil "github.com/sriniously/go-boilerplate/internal/testing"`

## Why a real database instead of mocks?

Mocks can't tell you that your SQL has a typo, that a constraint fires, or that
a migration is broken. For the repository layer, **a real Postgres is the only
honest test**. [Testcontainers](https://golang.testcontainers.org/) makes this
cheap: each test run gets a fresh container in a few seconds.

## Files

| File             | What it contains |
|------------------|------------------|
| `container.go`   | `TestDB` (pool + container + config). `SetupTestDB(t)` starts `postgres:15-alpine` with a random DB name, waits until ready, builds a test `Config`, connects with **5 retries**, runs **all migrations**, and registers `t.Cleanup` to terminate the container. `CleanupTestDB` for manual cleanup. |
| `server.go`      | `CreateTestServer(logger, testDB)`: a minimal `*server.Server` with Logger, DB and Config (New Relic disabled). **No Redis or jobs.** |
| `helpers.go`     | `SetupTest(t)`: DB + server + cleanup in one call. `MustMarshalJSON`, `ProjectRoot(t)` (walks up to `go.mod`), and generic `Ptr[T](v)` for optional fields |
| `transaction.go` | `WithTransaction` (commit on success) and `WithRollbackTransaction` (**always rolls back**, so tests leave no data behind) |
| `assertions.go`  | `AssertTimestampsValid`, `AssertValidUUID`, `AssertEqualExceptTime` (compares structs ignoring `time.Time` fields), `AssertStringContains` |

## Example test

```go
package repository_test

import (
    "context"
    "testing"

    "github.com/stretchr/testify/require"
    testutil "github.com/sriniously/go-boilerplate/internal/testing"
    "github.com/sriniously/go-boilerplate/internal/repository"
)

func TestTodoRepository_Create(t *testing.T) {
    _, srv, cleanup := testutil.SetupTest(t)
    defer cleanup()

    repo := repository.NewTodoRepository(srv)

    todo, err := repo.Create(context.Background(), "user_123", "write docs")
    require.NoError(t, err)

    testutil.AssertValidUUID(t, todo.ID)
    testutil.AssertTimestampsValid(t, todo)
    require.Equal(t, "write docs", todo.Title)
}
```

Run with `go test ./...`. **Docker must be running.**

## The testing pyramid for this codebase

| Layer        | Test type | Tooling |
|--------------|-----------|---------|
| repository   | integration (real DB) | this package |
| service      | unit (fake repo via interface) or integration | testify + this package |
| handler      | HTTP tests with `httptest` + Echo | `httptest.NewRecorder()` |
| whole system | a few end-to-end smoke tests | real server + HTTP client |

See [`../../docs/20-testing-strategy.md`](../../docs/20-testing-strategy.md).
