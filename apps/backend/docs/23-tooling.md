# 23 · Tooling: Taskfile, golangci-lint, go.mod, .gitignore

These files sit in the root of `apps/backend/`. They are not Go code, but a
production project depends on them just as much.

## `go.mod` / `go.sum`: dependencies

- `module github.com/sriniously/go-boilerplate` is the **import path prefix**
  for every internal package. If you rename the module (e.g. to your GitHub
  path), update every import, for example with
  `gofmt -r` / `go mod edit -module` + find-and-replace.
- `go.sum` holds **checksums** of every dependency, so builds are reproducible
  and tamper-evident. Always commit it.

Main direct dependencies and what they do:

| Module | Role |
|--------|------|
| `labstack/echo/v4` | HTTP router & middleware framework |
| `jackc/pgx/v5` | PostgreSQL driver + pool |
| `jackc/tern/v2` | SQL migrations |
| `jackc/pgx-zerolog` | pgx → zerolog adapter |
| `rs/zerolog` | structured logger |
| `knadh/koanf/v2` + `providers/env` | configuration loader |
| `joho/godotenv` | `.env` file loading |
| `go-playground/validator/v10` | struct-tag validation |
| `redis/go-redis/v9` | Redis client |
| `hibiken/asynq` | Redis-backed job queue |
| `clerk/clerk-sdk-go/v2` | auth token verification |
| `resend/resend-go/v2` | email API client |
| `newrelic/go-agent/v3` + integrations | APM, tracing, log forwarding |
| `google/uuid` | UUIDs |
| `pkg/errors` | stack-carrying errors (used by New Relic error reporting) |
| `stretchr/testify` | test assertions |
| `testcontainers/testcontainers-go` | Docker-based integration tests |

Useful commands:

```bash
go mod tidy        # add missing / remove unused deps
go mod verify      # check go.sum integrity
go list -m -u all  # see available upgrades
govulncheck ./...  # known vulnerabilities (go install golang.org/x/vuln/cmd/govulncheck@latest)
```

## `Taskfile.yml`: task runner

[Task](https://taskfile.dev) is a modern, cross-platform alternative to `make`.

| Task | Does |
|------|------|
| `task help` | list all tasks |
| `task run` | `go run ./cmd/go-boilerplate` |
| `task migrations:new name=x` | `tern new` in the migrations folder |
| `task migrations:up` | confirm prompt, then `tern migrate` using `BOILERPLATE_DB_DSN` |
| `task tidy` | `go fmt`, `go mod tidy`, `go mod verify` |

Why a task runner? So that every developer and CI runs **the exact same
commands**, and new team members discover them with `task help`.

## `.golangci.yml`: linting

[golangci-lint](https://golangci-lint.run) runs dozens of linters at once.
Highlights from this config:

| Linter | Catches |
|--------|---------|
| `errcheck` | ignored errors |
| `govet` (+ strict `shadow`) | suspicious code, shadowed variables |
| `staticcheck` | bugs, simplifications, deprecated APIs |
| `gosec` | security issues |
| `errorlint` | wrong error comparisons (`==` instead of `errors.Is`) |
| `bodyclose`, `sqlclosecheck`, `rowserrcheck` | leaked HTTP bodies / DB rows |
| `noctx` | HTTP requests without context |
| `funlen`, `gocognit`, `cyclop` | functions that are too long or too complex |
| `mnd` | magic numbers |
| `gochecknoglobals` | package-level variables |
| `exhaustive` | `switch` on enums missing cases |

```bash
golangci-lint run ./...
```

> The existing code doesn't pass every linter (e.g. `gochecknoglobals` will flag
> `job.emailClient`). Strict configs are often adopted gradually. Fix issues in
> files you touch.

## `.gitignore`

Ignores binaries, coverage output, `tmp/`, `go.work`, and most importantly
**`.env`**, so secrets don't get committed.

## `.env.sample`

The **template** for `.env`, with fake values. When you add a config key,
add it here too. It is documentation that new developers actually read.

## Suggested CI pipeline

```yaml
- go mod verify
- golangci-lint run ./...
- go test -race -cover ./...
- govulncheck ./...
- go build ./cmd/go-boilerplate
```
