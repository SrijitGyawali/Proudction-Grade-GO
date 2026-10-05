# 02 · Architecture diagram

GitHub renders these Mermaid diagrams automatically.

## Runtime components

```mermaid
flowchart LR
    client([Browser / Mobile / Frontend]) -->|HTTPS + Clerk JWT| api

    subgraph api[Go API process]
        direction TB
        router[Echo router + middleware] --> handlers[Handlers]
        handlers --> services[Services]
        services --> repos[Repositories]
        services --> jobclient[Asynq client]
        worker[Asynq worker] --> email[lib/email]
    end

    repos -->|pgxpool| pg[(PostgreSQL)]
    jobclient -->|enqueue| redis[(Redis)]
    redis -->|dequeue| worker
    email -->|HTTP API| resend[[Resend]]
    router -.->|verify token| clerk[[Clerk]]
    api -.->|traces, logs, events| nr[[New Relic]]
```

## Package dependency graph (who imports whom)

```mermaid
flowchart TD
    main[cmd/go-boilerplate] --> config
    main --> logger
    main --> database
    main --> server
    main --> repository
    main --> service
    main --> handler
    main --> router

    router --> middleware
    router --> handler
    handler --> service
    handler --> middleware
    handler --> validation
    service --> repository
    service --> job[lib/job]
    repository --> server
    middleware --> errs
    middleware --> sqlerr
    sqlerr --> errs
    validation --> errs
    server --> database
    server --> job
    server --> config
    job --> email[lib/email]
    database --> config
    database --> logger
    logger --> config
```

Notice: arrows only go **downwards**. `errs`, `config` and `model` are leaves
that many packages depend on, and they depend on almost nothing.

## Layers as an onion

```
┌───────────────────────────────────────────────────────────┐
│  main.go  (composition root: builds and wires everything)  │
│ ┌───────────────────────────────────────────────────────┐ │
│ │  router + middleware   (HTTP concerns, cross-cutting)  │ │
│ │ ┌───────────────────────────────────────────────────┐ │ │
│ │ │  handler + validation   (HTTP ⇄ Go translation)    │ │ │
│ │ │ ┌───────────────────────────────────────────────┐ │ │ │
│ │ │ │  service   (business rules)                    │ │ │ │
│ │ │ │ ┌───────────────────────────────────────────┐ │ │ │ │
│ │ │ │ │  repository   (Go ⇄ SQL translation)       │ │ │ │ │
│ │ │ │ │ ┌───────────────────────────────────────┐ │ │ │ │ │
│ │ │ │ │ │  model   (plain data structs)          │ │ │ │ │ │
│ │ │ │ │ └───────────────────────────────────────┘ │ │ │ │ │
│ │ │ │ └───────────────────────────────────────────┘ │ │ │ │
│ │ │ └───────────────────────────────────────────────┘ │ │ │
│ │ └───────────────────────────────────────────────────┘ │ │
│ └───────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
   infra on the side: config · logger · database · server · lib · errs · sqlerr
```

---

Next: [03 · Startup flow](03-startup-flow.md)
