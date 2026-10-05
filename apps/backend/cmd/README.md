# `cmd/` — where programs start

## What is this folder?

In Go projects, `cmd/` holds the **entry points** of the project. Each sub-folder
inside `cmd/` is one runnable program (one binary), and each one has its own
`package main` with a `func main()`.

```
cmd/
└── go-boilerplate/     <- one program: the HTTP API server
    └── main.go
```

## Why do we need it?

A real backend usually ends up with more than one program that share the same
code. For example:

| Folder (example)       | What it would do                                    |
|------------------------|-----------------------------------------------------|
| `cmd/go-boilerplate/`  | The API server (exists today)                       |
| `cmd/worker/`          | A separate process that only runs background jobs  |
| `cmd/migrate/`         | A small CLI that only runs database migrations      |
| `cmd/seed/`            | A script that fills the database with test data     |

All of them would import the same packages from `internal/`. Only the
"wiring" in `main.go` would be different.

## The golden rule

> `main.go` should be **thin**. It wires things together; it does not contain
> business logic.

Look at [`go-boilerplate/main.go`](go-boilerplate/main.go): it only loads
config, builds objects, starts the server and waits for shutdown. Every real
decision lives in `internal/`.

If you ever catch yourself writing an `if user.IsAdmin` inside `main.go`, that
code belongs in a service.

## How to run it

```bash
cd apps/backend
go run ./cmd/go-boilerplate      # or: task run
go build -o bin/api ./cmd/go-boilerplate
```

Run it **from `apps/backend`**. The server reads `static/` and
`templates/` using relative paths, so the working directory matters.

## Read next

- [`go-boilerplate/README.md`](go-boilerplate/README.md): a line-by-line walk through `main.go`
- [`../docs/03-startup-flow.md`](../docs/03-startup-flow.md): the full boot sequence
