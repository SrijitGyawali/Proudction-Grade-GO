# 21 · Adding a feature end-to-end: Todos

This chapter is a **recipe**. Follow it once and you will have touched every
layer. The code is illustrative. Adapt names as you like.

Goal: authenticated users can create, list and delete their todos.

```
POST   /api/v1/todos        create
GET    /api/v1/todos        list (paginated)
DELETE /api/v1/todos/:id    delete
```

## Step 1: Migration

```bash
task migrations:new name=create_todos
```

```sql
-- internal/database/migrations/002_create_todos.sql
CREATE TABLE todos (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     TEXT NOT NULL,
    title       TEXT NOT NULL,
    completed   BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_todos_user_id ON todos (user_id);

---- create above / drop below ----

DROP TABLE IF EXISTS todos;
```

```bash
task migrations:up
```

## Step 2: Model + DTOs

```go
// internal/model/todo.go
package model

type Todo struct {
    Base
    UserID    string `json:"userId"    db:"user_id"`
    Title     string `json:"title"     db:"title"`
    Completed bool   `json:"completed" db:"completed"`
}
```

```go
// internal/model/todo_dto.go
package model

import "github.com/go-playground/validator/v10"

var validate = validator.New()

type CreateTodoRequest struct {
    Title string `json:"title" validate:"required,min=1,max=200"`
}
func (r *CreateTodoRequest) Validate() error { return validate.Struct(r) }

type ListTodosRequest struct {
    Page  int `query:"page"  validate:"omitempty,min=1"`
    Limit int `query:"limit" validate:"omitempty,min=1,max=100"`
}
func (r *ListTodosRequest) Validate() error { return validate.Struct(r) }

type DeleteTodoRequest struct {
    ID string `param:"id" validate:"required,uuid"`
}
func (r *DeleteTodoRequest) Validate() error { return validate.Struct(r) }
```

## Step 3: Repository

```go
// internal/repository/todo.go
package repository

type TodoRepository struct{ server *server.Server }

func NewTodoRepository(s *server.Server) *TodoRepository { return &TodoRepository{server: s} }

func (r *TodoRepository) Create(ctx context.Context, userID, title string) (*model.Todo, error) {
    rows, err := r.server.DB.Pool.Query(ctx,
        `INSERT INTO todos (user_id, title) VALUES ($1, $2) RETURNING *`, userID, title)
    if err != nil {
        return nil, fmt.Errorf("insert todo: %w", err)
    }
    return pgx.CollectOneRow(rows, pgx.RowToAddrOfStructByName[model.Todo])
}

func (r *TodoRepository) ListByUser(ctx context.Context, userID string, limit, offset int) ([]model.Todo, int, error) {
    var total int
    if err := r.server.DB.Pool.QueryRow(ctx,
        `SELECT count(*) FROM todos WHERE user_id = $1`, userID).Scan(&total); err != nil {
        return nil, 0, fmt.Errorf("count todos: %w", err)
    }
    rows, err := r.server.DB.Pool.Query(ctx,
        `SELECT * FROM todos WHERE user_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3`,
        userID, limit, offset)
    if err != nil {
        return nil, 0, fmt.Errorf("list todos: %w", err)
    }
    todos, err := pgx.CollectRows(rows, pgx.RowToStructByName[model.Todo])
    return todos, total, err
}

func (r *TodoRepository) Delete(ctx context.Context, userID, id string) error {
    tag, err := r.server.DB.Pool.Exec(ctx,
        `DELETE FROM todos WHERE id = $1 AND user_id = $2`, id, userID)
    if err != nil {
        return fmt.Errorf("delete todo: %w", err)
    }
    if tag.RowsAffected() == 0 {
        return fmt.Errorf("table:todos: %w", pgx.ErrNoRows) // → 404 "Todo not found"
    }
    return nil
}
```

Register it: `Repositories{ Todo: NewTodoRepository(s) }`.

## Step 4: Service

```go
// internal/service/todo.go
package service

type TodoService struct {
    server *server.Server
    repo   *repository.TodoRepository
}

func NewTodoService(s *server.Server, r *repository.TodoRepository) *TodoService {
    return &TodoService{server: s, repo: r}
}

func (s *TodoService) Create(ctx context.Context, userID string, req *model.CreateTodoRequest) (*model.Todo, error) {
    return s.repo.Create(ctx, userID, strings.TrimSpace(req.Title))
}

func (s *TodoService) List(ctx context.Context, userID string, req *model.ListTodosRequest) (*model.PaginatedResponse[model.Todo], error) {
    page, limit := max(req.Page, 1), req.Limit
    if limit == 0 {
        limit = 20
    }
    todos, total, err := s.repo.ListByUser(ctx, userID, limit, (page-1)*limit)
    if err != nil {
        return nil, err
    }
    return &model.PaginatedResponse[model.Todo]{
        Data: todos, Page: page, Limit: limit, Total: total,
        TotalPages: (total + limit - 1) / limit,
    }, nil
}

func (s *TodoService) Delete(ctx context.Context, userID, id string) error {
    return s.repo.Delete(ctx, userID, id)
}
```

Register it in `NewServices`: `Todo: NewTodoService(s, repos.Todo)`.

## Step 5: Handler

```go
// internal/handler/todo.go
package handler

type TodoHandler struct {
    Handler
    svc *service.TodoService
}

func NewTodoHandler(s *server.Server, svc *service.TodoService) *TodoHandler {
    return &TodoHandler{Handler: NewHandler(s), svc: svc}
}

func (h *TodoHandler) Create(c echo.Context, req *model.CreateTodoRequest) (*model.Todo, error) {
    return h.svc.Create(c.Request().Context(), middleware.GetUserID(c), req)
}

func (h *TodoHandler) List(c echo.Context, req *model.ListTodosRequest) (*model.PaginatedResponse[model.Todo], error) {
    return h.svc.List(c.Request().Context(), middleware.GetUserID(c), req)
}

func (h *TodoHandler) Delete(c echo.Context, req *model.DeleteTodoRequest) error {
    return h.svc.Delete(c.Request().Context(), middleware.GetUserID(c), req.ID)
}
```

Register it in `NewHandlers`: `Todo: NewTodoHandler(s, services.Todo)`.

## Step 6: Routes

```go
// internal/router/todo.go
package router

func registerTodoRoutes(g *echo.Group, h *handler.Handlers, auth *middleware.AuthMiddleware) {
    t := g.Group("/todos", auth.RequireAuth)
    t.POST("", handler.Handle(h.Todo.Handler, h.Todo.Create, http.StatusCreated, &model.CreateTodoRequest{}))
    t.GET("", handler.Handle(h.Todo.Handler, h.Todo.List, http.StatusOK, &model.ListTodosRequest{}))
    t.DELETE("/:id", handler.HandleNoContent(h.Todo.Handler, h.Todo.Delete, http.StatusNoContent, &model.DeleteTodoRequest{}))
}
```

and in `NewRouter`:

```go
v1 := router.Group("/api/v1")
registerTodoRoutes(v1, h, middlewares.Auth)
```

## Step 7: Contract & docs

Add `ZTodo`, `ZCreateTodoRequest` to `packages/zod`, a `todoContract` to
`packages/openapi/src/contracts`, run the generator, and check `/docs`.

## Step 8: Tests

- repository integration test (Testcontainers)
- service test for pagination math
- handler test for 400 on empty title

## The file checklist

```
internal/database/migrations/002_create_todos.sql   (new)
internal/model/todo.go, todo_dto.go                 (new)
internal/repository/todo.go                         (new) + repositories.go (register)
internal/service/todo.go                            (new) + services.go (register)
internal/handler/todo.go                            (new) + handlers.go (register)
internal/router/todo.go                             (new) + router.go (call)
packages/zod/src/todo.ts, packages/openapi/src/contracts/todo.ts
```

Every feature follows **the same shape**. That is what makes the codebase scale
to many developers.
