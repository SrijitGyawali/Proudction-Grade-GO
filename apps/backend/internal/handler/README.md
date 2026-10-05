# `internal/handler/` — the HTTP layer (controllers)

## Job of this package

A handler is the **translator between HTTP and your business logic**:

```
HTTP request ──► [ handler ] ──► service method ──► result ──► [ handler ] ──► HTTP response
                  parse JSON                                    write JSON
                  validate                                      pick status code
                  read user id
```

A handler should **not** contain business rules or SQL. If you delete every
handler, the services should still make sense on their own.

## Files

| File          | What it contains |
|---------------|------------------|
| `handlers.go` | The `Handlers` struct (one field per handler group) and `NewHandlers(server, services)`, the single place where all handlers are built |
| `base.go`     | The shared machinery: base `Handler`, the generic `Handle` / `HandleNoContent` / `HandleFile` wrappers, and the `ResponseHandler` strategies (JSON, no content, file) |
| `health.go`   | `HealthHandler.CheckHealth`: `GET /status`, pings Postgres and Redis |
| `openapi.go`  | `OpenAPIHandler.ServeOpenAPIUI`: `GET /docs`, serves `static/openapi.html` |

## `base.go`: the most important file here

It uses **Go generics** to remove boilerplate from every endpoint:

```go
type HandlerFunc[Req validation.Validatable, Res any] func(c echo.Context, req Req) (Res, error)

func Handle[Req validation.Validatable, Res any](h Handler, fn HandlerFunc[Req, Res], status int, req Req) echo.HandlerFunc
```

When you wrap your function with `Handle(...)`, you get all of this for free:

1. **Logging** with request-scoped fields (`operation`, `method`, `path`, `route`)
2. **Bind + validate** the request into your `Req` struct (`validation.BindAndValidate`)
3. **Timing** of the validation step and the handler step
4. **New Relic attributes** (`validation.status`, `handler.duration_ms`, …) and `NoticeError` on failure
5. **Writing the response** with the right strategy (JSON, 204, or file download)

### Strategy pattern: `ResponseHandler`

```go
type ResponseHandler interface {
    Handle(c echo.Context, result interface{}) error   // how to write the response
    GetOperation() string                              // label for logs
    AddAttributes(txn *newrelic.Transaction, result interface{})
}
```

`JSONResponseHandler`, `NoContentResponseHandler` and `FileResponseHandler`
implement it. `handleRequest` does not care which one it gets. Adding, say, a
`CSVResponseHandler` means writing one small type, with no changes to the core.

## What a real endpoint looks like

```go
// request DTO, lives near the handler (or in model/)
type CreateTodoRequest struct {
    Title string `json:"title" validate:"required,min=1,max=200"`
}
func (r *CreateTodoRequest) Validate() error { return validator.New().Struct(r) }

// handler
type TodoHandler struct {
    Handler
    todoService *service.TodoService
}

func (h *TodoHandler) Create(c echo.Context, req *CreateTodoRequest) (*model.Todo, error) {
    userID := middleware.GetUserID(c)
    return h.todoService.Create(c.Request().Context(), userID, req.Title)
}

// router
todos.POST("", handler.Handle(h.Todo.Handler, h.Todo.Create, http.StatusCreated, &CreateTodoRequest{}))
```

The handler body is **two lines**. Everything else is handled by `Handle`.

> ⚠️ The `req` you pass to `Handle(...)` is created **once**, when the route is
> registered, and that same pointer is reused for every request. Under
> concurrent traffic two requests can bind into the same struct. A safer
> variant builds a fresh value per request, e.g. pass a factory
> `func() Req { return &CreateTodoRequest{} }`. See [`../../docs/25-gotchas.md`](../../docs/25-gotchas.md).

## `health.go`

- Pings **Postgres** and **Redis**, each with a 5 s timeout, and measures response time.
- Postgres down → overall `unhealthy` → **HTTP 503**. Load balancers and
  Kubernetes use this to stop sending traffic.
- Redis down → reported as unhealthy in `checks.redis`, but the overall status
  **stays healthy**, because Redis is treated as non-critical.
- Records a `HealthCheckError` custom event in New Relic for each failure.

## `openapi.go`

Reads `static/openapi.html` from disk on each request (with `Cache-Control:
no-cache`) and returns it. That HTML loads the **Scalar** API reference UI,
which fetches `/static/openapi.json`.

## Rules of thumb

- Handlers take `echo.Context`; **services never do**. Pass
  `c.Request().Context()` down instead.
- Get the user with `middleware.GetUserID(c)`. Do not parse tokens here.
- Return errors; do not write error JSON yourself. The global error handler does it.
