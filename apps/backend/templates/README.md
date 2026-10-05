# `templates/` — server-side rendered templates

## What is this folder for?

Text/HTML templates that the Go backend **fills with data at runtime**. Today it
contains email templates only:

```
templates/
└── emails/
    └── welcome.html
```

Other things that could live here later: PDF/invoice templates, SMS text
templates, server-rendered HTML pages.

## Why outside `internal/`?

Like `static/`, these are **assets, not Go code**. Designers or frontend devs
can update them without touching Go. They are loaded from disk at runtime with
a relative path (`templates/emails/<name>.html`), so:

- run the server from `apps/backend`, and
- copy this folder into your Docker image (or switch to `//go:embed`).

## `static/` vs `templates/`

| `static/`                                | `templates/`                                   |
|------------------------------------------|------------------------------------------------|
| Sent to the client **unchanged**         | **Processed** by Go (`html/template`) first    |
| Publicly reachable at `/static/...`      | Never served directly; private to the server  |
| e.g. `openapi.json`                      | e.g. `welcome.html` with `{{.UserFirstName}}`  |

See [`emails/README.md`](emails/README.md).
