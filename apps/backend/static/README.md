# `static/` — files served as-is over HTTP

## What is in here?

| File           | What it is |
|----------------|------------|
| `openapi.json` | The **OpenAPI 3** specification of the API: every path, method, request and response schema |
| `openapi.html` | A tiny HTML page that loads the [Scalar](https://scalar.com) API-reference UI from a CDN and points it at `/static/openapi.json` |

## How they are served

In `internal/router/system.go`:

```go
r.Static("/static", "static")             // GET /static/openapi.json → ./static/openapi.json
r.GET("/docs", h.OpenAPI.ServeOpenAPIUI)  // GET /docs → contents of ./static/openapi.html
```

Open **http://localhost:8080/docs** while the server runs to browse and try the API.

## Where does `openapi.json` come from? (Don't edit it by hand!)

It is **generated** from TypeScript contracts in the monorepo:

```
packages/zod/src/*.ts              Zod schemas (shapes of requests/responses)
        │
packages/openapi/src/contracts/*.ts   ts-rest contracts (path + method + schemas)
        │  bun run gen   (packages/openapi → src/gen.ts)
        ▼
packages/openapi/openapi.json
apps/backend/static/openapi.json   ← written here too
```

This is **contract-first** design. The frontend and the docs share one source of
truth, and the frontend gets a fully typed API client from the same contracts.

**Workflow when you add an endpoint:**
1. Add the Zod schemas in `packages/zod`.
2. Add the route to a contract in `packages/openapi/src/contracts`.
3. Run the generator → `static/openapi.json` updates.
4. Implement the Go handler with the same path and shapes.

## Why is this folder outside `internal/`?

`internal/` is for **Go code**. `static/` holds **runtime assets** that the binary
reads from disk. Because paths are relative (`"static"`), the server must be
started from `apps/backend`, and a Docker image must copy this folder next to
the binary. Alternatively, use `//go:embed static` to bake it into the binary.

## Security note

Everything in `static/` is **publicly downloadable**. Never put secrets, env
files or private data here. If the API docs should not be public in production,
put `/docs` and `/static/openapi.json` behind auth or disable them by env.
