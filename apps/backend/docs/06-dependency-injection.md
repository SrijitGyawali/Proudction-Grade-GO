# 06 · Dependency injection, the Go way

## What is DI?

"Dependency injection" sounds fancy but just means:

> **A component receives the things it needs instead of creating them itself.**

```go
// ❌ creates its own dependency: hard to test, hidden coupling
func NewTodoService() *TodoService {
    pool, _ := pgxpool.New(context.Background(), os.Getenv("DB_URL"))
    return &TodoService{repo: &TodoRepository{pool: pool}}
}

// ✅ receives it
func NewTodoService(s *server.Server, repo *repository.TodoRepository) *TodoService {
    return &TodoService{server: s, repo: repo}
}
```

## How this codebase does it: constructor injection + aggregate structs

Each layer has an **aggregate struct** and **one constructor** that builds the
whole layer:

| Layer | Aggregate | Constructor | Receives |
|-------|-----------|-------------|----------|
| infra | `server.Server` | `server.New(cfg, log, nr)` | config, logger |
| data | `repository.Repositories` | `NewRepositories(srv)` | server |
| business | `service.Services` | `NewServices(srv, repos)` | server, repos |
| HTTP | `handler.Handlers` | `NewHandlers(srv, services)` | server, services |
| HTTP | `middleware.Middlewares` | `NewMiddlewares(srv)` | server |
| routing | `*echo.Echo` | `router.NewRouter(srv, handlers, services)` | everything above |

`main.go` is the **composition root**, the only place that knows how
everything fits together:

```go
repos    := repository.NewRepositories(srv)
services, _ := service.NewServices(srv, repos)
handlers := handler.NewHandlers(srv, services)
r        := router.NewRouter(srv, handlers, services)
```

## Why no DI framework (wire, fx, dig)?

- It is a few lines, readable top to bottom.
- Compile-time safety: forget a dependency and it doesn't compile.
- No reflection, no generated code, no magic to debug.

Frameworks help when you have hundreds of components. Most services never get
there.

## Making it testable: depend on interfaces (where it matters)

Today services depend on concrete repository types. When you want unit tests
for a service without a DB, introduce a small interface **in the consumer**
(the service package):

```go
// service/todo.go
type todoStore interface {
    Create(ctx context.Context, userID, title string) (*model.Todo, error)
    CountByUser(ctx context.Context, userID string) (int, error)
}

type TodoService struct{ repo todoStore }
```

`*repository.TodoRepository` satisfies it **implicitly** (Go interfaces need no
`implements` keyword). In tests:

```go
type fakeStore struct{ count int }
func (f *fakeStore) CountByUser(context.Context, string) (int, error) { return f.count, nil }
func (f *fakeStore) Create(context.Context, string, string) (*model.Todo, error) { return &model.Todo{}, nil }

svc := &TodoService{repo: &fakeStore{count: 10}}
_, err := svc.Create(ctx, "u1", "x")   // expect TODO_LIMIT_REACHED
```

Go proverb: **"Accept interfaces, return structs."**

## The `*server.Server` trade-off

Passing the whole `Server` is convenient: one parameter, access to everything.
The cost is that it hides what a component *really* needs. If a repository
only needs the pool, `NewTodoRepository(pool *pgxpool.Pool)` documents that
better. Both styles are fine. Pick one and be consistent.

---

Next: [07 · Configuration](07-configuration.md)
