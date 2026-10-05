# `internal/model/` — the shapes of your data

## Job of this package

Define the **domain structs**: the nouns of your app (User, Todo, Invoice…)
plus shared building blocks. Every layer imports `model`, and `model` imports
(almost) nothing. It sits at the **bottom of the dependency graph**.

## Files

| File      | What it contains |
|-----------|------------------|
| `base.go` | Reusable embedded structs: `BaseWithId` (UUID `id`), `BaseWithCreatedAt`, `BaseWithUpdatedAt`, `Base` (all three), and the generic `PaginatedResponse[T]` |

## Struct embedding (Go's "composition over inheritance")

```go
type Base struct {
    BaseWithId          // ID        uuid.UUID `json:"id"        db:"id"`
    BaseWithCreatedAt   // CreatedAt time.Time `json:"createdAt" db:"created_at"`
    BaseWithUpdatedAt   // UpdatedAt time.Time `json:"updatedAt" db:"updated_at"`
}

type Todo struct {
    model.Base                       // ← fields are "promoted": todo.ID, todo.CreatedAt
    Title  string `json:"title"  db:"title"`
    UserID string `json:"userId" db:"user_id"`
}
```

Go has no classes or inheritance. Embedding **copies the fields in**, so every
table gets `id / created_at / updated_at` consistently.

## The two tags

| Tag    | Used by | Convention |
|--------|---------|------------|
| `json` | Echo when writing responses | **camelCase** (`createdAt`), what JS clients expect |
| `db`   | pgx `RowToStructByName` when scanning rows | **snake_case** (`created_at`), matching Postgres columns |

## `PaginatedResponse[T]` (generics)

```go
type PaginatedResponse[T interface{}] struct {
    Data       []T `json:"data"`
    Page       int `json:"page"`
    Limit      int `json:"limit"`
    Total      int `json:"total"`
    TotalPages int `json:"totalPages"`
}

// usage
return &model.PaginatedResponse[model.Todo]{Data: todos, Page: 1, Limit: 20, Total: 57, TotalPages: 3}
```

One type works for every list endpoint, so the frontend always gets the same
pagination shape.

## Organising models as the app grows

```
model/
├── base.go
├── todo/            ← one sub-package per domain (or one file per domain)
│   ├── todo.go      entity struct
│   └── dto.go       CreateTodoRequest, UpdateTodoRequest, TodoResponse
└── user/
```

**Entity** = what is stored. **DTO** (Data Transfer Object) = what crosses the
API boundary. Keeping them separate lets you hide internal columns
(`password_hash`) and shape responses freely.
