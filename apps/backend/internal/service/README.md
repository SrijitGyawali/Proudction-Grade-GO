# `internal/service/` — business logic (the "brain")

## Job of this package

Implement **use cases**: what your product actually *does*. For example:
"register a user", "create a todo", "invite a teammate", "close an invoice".

A service:
- enforces **business rules** ("free plan users can have max 10 todos"),
- **orchestrates** repositories, external clients and background jobs,
- decides **transaction boundaries**,
- returns domain results or `*errs.HTTPError` for expected failures.

A service **does not know about HTTP**. No `echo.Context`, no status codes
(except via `errs`), no JSON. That way the same service could be called from an
HTTP handler, a CLI command, a job worker, or a test.

## Files

| File          | What it contains |
|---------------|------------------|
| `services.go` | `Services` struct (one field per service) and `NewServices(server, repos)`: the single place where services are built and wired |
| `auth.go`     | `AuthService`: on creation it calls `clerk.SetKey(cfg.Auth.SecretKey)`, which configures the **Clerk SDK globally** so `RequireAuth` can verify tokens |

`Services` also exposes `Job` (the `JobService` from the server), so any
service can enqueue background tasks.

## Why does `AuthService` exist if it's almost empty?

`clerk.SetKey` must run **once at startup, before the first request**.
Constructing `AuthService` in `NewServices` guarantees that. It is also the
natural home for future auth logic: syncing Clerk users into your DB, checking
roles, handling webhooks.

## What a real service looks like

```go
type TodoService struct {
    server   *server.Server
    todoRepo *repository.TodoRepository
}

func (s *TodoService) Create(ctx context.Context, userID, title string) (*model.Todo, error) {
    // 1. business rule
    count, err := s.todoRepo.CountByUser(ctx, userID)
    if err != nil {
        return nil, err
    }
    if count >= 10 {
        code := "TODO_LIMIT_REACHED"
        return nil, errs.NewBadRequestError("Free plan allows 10 todos", true, &code, nil, nil)
    }

    // 2. persistence
    todo, err := s.todoRepo.Create(ctx, userID, title)
    if err != nil {
        return nil, err
    }

    // 3. side effects → background job (don't block the request)
    // task, _ := job.NewTodoCreatedTask(todo.ID)
    // _, _ = s.server.Job.Client.EnqueueContext(ctx, task)

    return todo, nil
}
```

Then wire it:

```go
type Services struct {
    Auth *AuthService
    Job  *job.JobService
    Todo *TodoService
}
// in NewServices:
Todo: NewTodoService(s, repos.Todo),
```

## Why does `NewServices` return `(*Services, error)`?

Some services may need fallible setup, such as loading a key file or
validating a remote config. The signature is ready for that, and `main.go`
already handles the error.

## Rules of thumb

- **First parameter is `ctx context.Context`.** Pass it everywhere downstream.
- **Thin handlers, fat services, dumb repositories.**
- Return `errs.*` for *expected* failures (not found, forbidden, limit reached);
  wrap and return everything else.
- If a method needs `echo.Context`, the logic is in the wrong layer.
