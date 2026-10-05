# `internal/` — the private heart of the application

## Why is it called `internal`?

`internal` is a **special folder name in Go**, and the compiler enforces it:

> Code inside `internal/` can only be imported by code rooted at the parent of
> `internal/`.

So `github.com/sriniously/go-boilerplate/internal/service` can be imported by
`cmd/go-boilerplate/main.go`, but **no other Go module in the world** can import
it. This lets you change anything in here freely without breaking someone
else's code. Everything that is "our app" lives here.

(The opposite convention is `pkg/`, for code you *want* others to import. This
project has no `pkg/`, which is the right choice for an application.)

## The folders, grouped by role

```
internal/
│
├── ── BOOTSTRAP / INFRASTRUCTURE ──────────────────────────
│   ├── config/        read & validate env vars into typed structs
│   ├── logger/        zerolog + New Relic setup
│   ├── database/      Postgres pool + migrations
│   └── server/        the "container" holding DB, Redis, Jobs, Config, Logger
│
├── ── HTTP LAYER (the outside world talks to us here) ─────
│   ├── router/        URL → handler mapping + middleware order
│   ├── middleware/    code that runs around every request (auth, logs, tracing…)
│   ├── handler/       parse request → call service → write response
│   └── validation/    bind JSON + validate struct tags
│
├── ── BUSINESS LAYER ──────────────────────────────────────
│   ├── service/       the rules of your product (use cases)
│   └── model/         the shapes of your data (structs)
│
├── ── DATA LAYER ──────────────────────────────────────────
│   └── repository/    SQL queries; the only place that talks to Postgres
│
├── ── ERRORS ──────────────────────────────────────────────
│   ├── errs/          the one error type the API returns (HTTPError)
│   └── sqlerr/        turns Postgres errors into friendly HTTPErrors
│
├── ── SHARED HELPERS ──────────────────────────────────────
│   └── lib/
│       ├── email/     send emails via Resend using HTML templates
│       ├── job/       background jobs via Asynq + Redis
│       └── utils/     tiny generic helpers
│
└── ── TESTS ───────────────────────────────────────────────
    └── testing/       spin up a real Postgres in Docker for integration tests
```

## The one rule that keeps this clean

**Dependencies point inward/downward only.**

```
router → middleware → handler → service → repository → database
                         │          │
                         └── model ─┘
```

- A **handler** may call a **service**. A service must **never** import `handler`
  or `echo`.
- A **service** may call a **repository**. A repository must **never** call a
  service.
- **model** is imported by everyone and imports nobody (except std lib + uuid).

If you follow this, you can:
- unit-test a service without HTTP,
- swap Postgres for something else by only touching `repository/`,
- add a gRPC or CLI front-end later that reuses all the services.

## Every folder has its own README

Open any sub-folder to see what each file does and why it exists.
