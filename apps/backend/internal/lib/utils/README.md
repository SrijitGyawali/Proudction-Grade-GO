# `internal/lib/utils/` — tiny generic helpers

## Files

| File       | What it contains |
|------------|------------------|
| `utils.go` | `PrintJSON(v)`: marshals any value with indentation and prints it to stdout. Handy while debugging. |

## When to put something here

Only **small, pure, dependency-free** helpers that are needed in more than one
place and do not belong to a specific domain:

- string helpers (slugify, truncate)
- pointer helpers (`Ptr[T](v T) *T`)
- slice helpers (`Map`, `Filter`, `Contains`)
- time helpers (`StartOfDay`)

## Warning: avoid the "utils junk drawer"

A `utils` package tends to grow forever and becomes a place nobody owns. Before
adding something, ask:

- Does it belong to a domain? → put it next to that domain (`service/`, `model/`).
- Does it wrap an external system? → make its own `lib/<name>/`.
- Is it only used once? → keep it private in the file that uses it.

## About `PrintJSON`

It uses `fmt.Println`, so it bypasses the structured logger. That is fine for
local debugging, but **remove calls before committing**. In real code, log with
the request logger: `middleware.GetLogger(c).Debug().Interface("payload", v).Msg("…")`.
