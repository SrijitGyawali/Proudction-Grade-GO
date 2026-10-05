# `internal/repository/` — the data-access layer

## Job of this package

Be the **only place in the app that knows SQL**. Repositories take Go values,
run queries against Postgres (`server.DB.Pool`), and return Go structs from
`model/`.

```
service ──(Go types)──► repository ──(SQL)──► Postgres
service ◄──(model.X)─── repository ◄──(rows)─ Postgres
```

## Files

| File              | What it contains |
|-------------------|------------------|
| `repositories.go` | The `Repositories` struct (currently empty: `struct{}`) and `NewRepositories(server)`. It is a **placeholder** ready for your first repository. |

## Why have a repository layer?

1. **Separation**: services read like business rules, not SQL strings.
2. **Swappability**: change a query, add caching, or move a table to another
   store without touching services.
3. **Testability**: services can depend on an interface; tests can pass a fake.
4. **One home per table**: all `todos` queries live in `todo.go`, so they are easy
   to review and to index properly.

## What a real repository looks like

```go
// repository/todo.go
type TodoRepository struct{ server *server.Server }

func NewTodoRepository(s *server.Server) *TodoRepository { return &TodoRepository{server: s} }

func (r *TodoRepository) Create(ctx context.Context, userID, title string) (*model.Todo, error) {
    const q = `
        INSERT INTO todos (user_id, title)
        VALUES (@user_id, @title)
        RETURNING *`
    rows, err := r.server.DB.Pool.Query(ctx, q, pgx.NamedArgs{"user_id": userID, "title": title})
    if err != nil {
        return nil, fmt.Errorf("insert todo: %w", err)
    }
    todo, err := pgx.CollectOneRow(rows, pgx.RowToAddrOfStructByName[model.Todo])
    if err != nil {
        return nil, fmt.Errorf("collect todo: %w", err)
    }
    return todo, nil
}
```

and register it:

```go
type Repositories struct { Todo *TodoRepository }

func NewRepositories(s *server.Server) *Repositories {
    return &Repositories{ Todo: NewTodoRepository(s) }
}
```

## Rules

- **Always parameterise** (`$1` or `@name`). Never build SQL with `fmt.Sprintf`
  and user input, because that is how SQL injection happens.
- **Always take `ctx context.Context`** as the first argument and pass it to pgx,
  so a cancelled request also cancels its query.
- **Wrap errors** with `%w` (`fmt.Errorf("get todo %s: %w", id, err)`). The
  global handler can still detect Postgres codes through `errors.As`.
- **Return `pgx.ErrNoRows`** (wrapped) for "not found". `sqlerr.HandleError`
  turns it into a 404. If the wrapped message contains `table:todos:` it even
  says "Todo not found".
- **No business rules here.** "Only the owner can delete" belongs in the service.
- **Transactions**: when one use case writes several tables, start a `pgx.Tx`
  in the service and pass it to repository methods that accept a `pgx.Tx`
  (or a small `DBTX` interface implemented by both pool and tx).
