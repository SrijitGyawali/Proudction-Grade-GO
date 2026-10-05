# Learning this codebase

This repository is a production-grade Go + TypeScript monorepo. On top of the
original boilerplate it contains a **complete study guide** for the Go backend.

## Start here

1. 📘 **[Backend learning guide](apps/backend/docs/README.md)**: 29 chapters, from
   the mental model to adding your first feature.
2. 📂 **Every backend folder has a `README.md`** explaining what each file does
   and why it exists. Start with [`apps/backend/internal/README.md`](apps/backend/internal/README.md).

## Quick map

```
apps/backend/
├── cmd/go-boilerplate/   entry point (main.go): wiring + graceful shutdown
├── internal/
│   ├── config/           env vars → typed, validated config
│   ├── logger/           zerolog + New Relic
│   ├── database/         pgx pool + tern migrations
│   ├── server/           container for DB, Redis, jobs, HTTP server
│   ├── router/           routes + middleware order
│   ├── middleware/       auth, logging, tracing, request id, errors
│   ├── handler/          HTTP ⇄ Go translation (generic Handle wrapper)
│   ├── validation/       bind + validate requests
│   ├── service/          business logic
│   ├── repository/       SQL
│   ├── model/            data structs
│   ├── errs/             API error type
│   ├── sqlerr/           Postgres error → friendly API error
│   ├── lib/              email (Resend), job (Asynq), utils
│   └── testing/          Testcontainers-based test toolkit
├── static/               OpenAPI spec + docs UI
├── templates/            email HTML templates
└── docs/                 📘 the learning guide
```

## Recommended path

| Day | Read | Do |
|-----|------|----|
| 1 | Chapters 01–06 | Run the server, hit `/status` and `/docs` |
| 2 | Chapters 07–13 | Write your first migration |
| 3 | Chapters 14–20 | Write one integration test |
| 4 | Chapter 21 | Build the Todo feature end-to-end |
| 5 | Chapters 22–29 | Pick 3 exercises from chapter 29 |

Original project: [sriniously/go-boilerplate](https://github.com/sriniously/go-boilerplate).
