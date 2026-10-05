# 22 · Go patterns used in this codebase

A tour of the Go language features and idioms you'll meet while reading the code.

## 1. Constructor functions (`NewXxx`)

Go has no constructors. By convention, `NewXxx(...) *Xxx` builds a ready-to-use
value.

```go
func NewHealthHandler(s *server.Server) *HealthHandler
```

## 2. Struct embedding (composition)

```go
type HealthHandler struct {
    Handler          // embedded: HealthHandler gets Handler's fields and methods
}
```

Used in `handler/` (shared base) and `model/` (shared `id`, `created_at`, `updated_at`).

## 3. Implicit interfaces

A type implements an interface just by having the methods. There is no
`implements` keyword.

```go
type Validatable interface { Validate() error }
// any request struct with Validate() error satisfies it
```

`*errs.HTTPError` satisfies the built-in `error` interface through its `Error()` method.

## 4. Generics (Go 1.18+)

```go
func Handle[Req validation.Validatable, Res any](h Handler, fn HandlerFunc[Req, Res], status int, req Req) echo.HandlerFunc
type PaginatedResponse[T interface{}] struct { Data []T ... }
func Ptr[T any](v T) *T { return &v }
```

One implementation, type-safe for every request/response type.

## 5. Higher-order functions & closures (middleware)

```go
func RequestID() echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {   // takes a func, returns a func
        return func(c echo.Context) error { ...; return next(c) }
    }
}
```

## 6. Strategy pattern via interface

`ResponseHandler` with `JSONResponseHandler` / `NoContentResponseHandler` /
`FileResponseHandler`. The algorithm (`handleRequest`) is fixed, and the
response writing is pluggable.

## 7. Composite pattern

`database.multiTracer` holds several tracers and forwards calls to each, so
pgx sees one tracer.

## 8. Anonymous interface type assertions

```go
if t, ok := tracer.(interface {
    TraceQueryStart(context.Context, *pgx.Conn, pgx.TraceQueryStartData) context.Context
}); ok { ... }
```

This checks "does this value have that method?" without declaring a named interface.

## 9. Error wrapping

```go
fmt.Errorf("failed to parse pgx pool config: %w", err)
errors.As(err, &httpErr)
errors.Is(err, pgx.ErrNoRows)
func (pe *Error) Unwrap() error { return pe.driverErr }   // makes the chain walkable
```

## 10. `context.Context` everywhere

Cancellation, deadlines and request-scoped values (`context.WithValue` for the
logger). The first parameter of any I/O function.

## 11. Goroutine + signal for lifecycle

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
go srv.Start()
<-ctx.Done()
```

## 12. `defer` for cleanup

```go
ctx, cancel := context.WithTimeout(...)
defer cancel()
defer tx.Rollback(ctx)   // no-op if already committed
```

## 13. Blank imports for side effects

```go
import _ "github.com/joho/godotenv/autoload"   // runs init() that loads .env
```

## 14. `//go:embed`

```go
//go:embed migrations/*.sql
var migrations embed.FS
```

Files are compiled into the binary.

## 15. Typed string constants as enums

```go
type Template string
const TemplateWelcome Template = "welcome"

type Code string
const UniqueViolation Code = "unique_violation"
```

## 16. Struct tags

```go
Port string `koanf:"port" validate:"required"`
ID   uuid.UUID `json:"id" db:"id"`
Page int `query:"page"`
```

Metadata that libraries read with reflection: config loading, validation, JSON,
DB scanning, request binding.

## 17. `internal/` visibility

The compiler stops other modules from importing your private packages.

## 18. Nil-safe optional dependencies

```go
if s.LoggerService != nil && s.LoggerService.GetApplication() != nil { ... }
```

The app works with or without New Relic.
