# `internal/router/` — URLs, route groups and the middleware chain

## Job of this package

Create the **Echo** instance and decide:

1. which **middleware** runs on every request, and in what order,
2. which **URL + method** maps to which **handler**,
3. which routes need **authentication** or other per-group middleware.

It is the **table of contents** of your API. To see what the backend exposes,
start here.

## Files

| File        | What it contains |
|-------------|------------------|
| `router.go` | `NewRouter(server, handlers, services) *echo.Echo`: builds middlewares, sets the global error handler, installs the global middleware chain (including the rate limiter), registers system routes, and creates the `/api/v1` group |
| `system.go` | `registerSystemRoutes`: infrastructure endpoints that aren't part of the product API |

## Routes today

| Method | Path        | Handler                         | Purpose |
|--------|-------------|---------------------------------|---------|
| GET    | `/status`   | `Health.CheckHealth`            | Health check for load balancers / uptime monitors |
| GET    | `/docs`     | `OpenAPI.ServeOpenAPIUI`        | Interactive API docs (Scalar UI) |
| GET    | `/static/*` | Echo static file server         | Serves files from `./static` (e.g. `openapi.json`) |

## Rate limiter

```go
echoMiddleware.RateLimiterWithConfig(echoMiddleware.RateLimiterConfig{
    Store: echoMiddleware.NewRateLimiterMemoryStore(rate.Limit(20)),   // 20 req/s per client IP
    DenyHandler: ... → log + New Relic event + 429 "Rate limit exceeded"
})
```

## Versioned API group

```go
router.Group("/api/v1")
```

This line creates the group but **does not keep the return value**, so it is a
placeholder. When you add features, capture it and register routes on it:

```go
func NewRouter(s *server.Server, h *handler.Handlers, services *service.Services) *echo.Echo {
    ...
    registerSystemRoutes(router, h)

    v1 := router.Group("/api/v1")
    registerTodoRoutes(v1, h, middlewares.Auth)
    return router
}

// router/todo.go
func registerTodoRoutes(g *echo.Group, h *handler.Handlers, auth *middleware.AuthMiddleware) {
    todos := g.Group("/todos", auth.RequireAuth)
    todos.GET("",        handler.Handle(h.Todo.Handler, h.Todo.List,   http.StatusOK,      &todo.ListRequest{}))
    todos.POST("",       handler.Handle(h.Todo.Handler, h.Todo.Create, http.StatusCreated, &todo.CreateRequest{}))
    todos.DELETE("/:id", handler.HandleNoContent(h.Todo.Handler, h.Todo.Delete, http.StatusNoContent, &todo.DeleteRequest{}))
}
```

**One `registerXxxRoutes` function per feature, one file per feature**, the same
pattern as `system.go`.

## Why version the API (`/v1`)?

Mobile apps and third-party clients can't all upgrade on the day you deploy.
When you need a breaking change, you add `/api/v2` and keep `/v1` alive until
old clients are gone.

## Why does `NewRouter` receive `services`?

It is passed in but not used today. It is handy if a route-level middleware
ever needs a service, e.g. "load the organisation and check membership".
