# 01 · The mental model of a production Go backend

Before looking at folders, get the **ideas** right. The folders are just these
ideas written down.

## Idea 1: A backend is a pipeline of translations

```
 bytes on the network
   → HTTP request              (router, middleware)
     → validated Go struct      (validation, handler)
       → business decision       (service)
         → SQL query               (repository)
           → rows on disk            (Postgres)
         ← Go structs              (repository)
       ← result or domain error  (service)
     ← JSON + status code       (handler, error handler)
   ← bytes on the network
```

Each layer translates from one "language" to the next, and **only that layer
speaks both languages**. Handlers speak HTTP and Go. Repositories speak Go and
SQL. Services speak only Go, the language of your *business*.

## Idea 2: Separate "what" from "how" from "where"

| Concern | Question it answers | Lives in |
|---------|---------------------|----------|
| **Transport** | *Where* did the request come from, and how do we reply? | `router`, `middleware`, `handler` |
| **Business** | *What* should happen? | `service`, `model` |
| **Persistence** | *How* is it stored? | `repository`, `database` |
| **Infrastructure** | What do we need to run at all? | `config`, `logger`, `server`, `lib` |

If you can explain a feature without saying "HTTP" or "SQL", that explanation
is your service code.

## Idea 3: Dependencies point one way

```
handler ──► service ──► repository ──► database
   │           │             │
   └───────────┴──► model ◄──┘
```

Lower layers **never import** higher layers. This one rule gives you
testability, replaceability and a codebase that can still change after two
years.

## Idea 4: Production = the boring stuff done right

Writing the happy path is about 20% of the work. A production service also
needs:

| Concern | Question | In this repo |
|---------|----------|--------------|
| Config | Can I change settings without recompiling? Does bad config fail loudly? | `internal/config` |
| Observability | When it breaks at 3 a.m., can I find out *why* from logs/traces alone? | `logger`, `middleware/tracing`, `context` |
| Errors | Do clients get consistent, safe errors? Do I get the real cause in logs? | `errs`, `sqlerr`, `GlobalErrorHandler` |
| Validation | Is every input checked before use? | `validation` |
| Security | Auth, rate limits, CORS, headers, secrets out of code? | `middleware`, router |
| Resilience | Timeouts? Retries? Does one slow dependency take everything down? | `server` timeouts, job retries |
| Lifecycle | Clean startup checks? Graceful shutdown on deploy? | `main.go`, `server.Shutdown` |
| Schema evolution | Can the DB change safely while running? | `database/migrations` |
| Async work | Are slow tasks kept off the request path? | `lib/job` |
| Testing | Can I prove it works against a real DB? | `internal/testing` |
| Docs | Can the frontend team use the API without asking me? | `static/openapi.json` |

This boilerplate gives you all of these **before you write a single feature**.
That is why it is "production grade".

## Idea 5: Every request carries a context

In Go, `context.Context` is passed as the first argument through **every** layer.
It carries:
- **cancellation**: if the client disconnects or a timeout hits, DB queries stop too,
- **deadlines**,
- **request-scoped values**: request id, logger, New Relic transaction.

Rule: *if a function does I/O, its first parameter is `ctx context.Context`.*

## Idea 6: Fail fast at startup, degrade gracefully at runtime

- **Startup**: missing config or DB unreachable → crash immediately with a clear
  message. A half-working server is worse than none.
- **Runtime**: a single request failing must **never** crash the process
  (`Recover` middleware), and an optional dependency being down (Redis) should
  degrade features, not kill the API.

## Idea 7: Explicit over magic

Go culture prefers code you can **read top to bottom**. There is no DI container,
no annotations, no reflection-driven routing. `main.go` builds everything by
hand. When you want to know "where does X come from?", you can always
*Go to Definition* and find it.

---

Next: [02 · Architecture diagram](02-architecture-diagram.md)
