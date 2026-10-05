# `internal/lib/job/` — background jobs with Asynq + Redis

## Why background jobs?

Some work is **slow**, **can fail**, or **does not need to finish before we
reply** to the user:

- sending emails / SMS / push notifications
- generating PDFs or reports
- calling slow third-party APIs
- cleanup tasks, retries, scheduled work

Doing these inside the HTTP request makes the API slow and fragile. Instead we
**enqueue a task** (a tiny JSON message) in Redis and return immediately. A
**worker** picks it up and processes it, with automatic **retries**.

```
 HTTP request                         Redis                       Worker (same process here)
──────────────                   ────────────────              ─────────────────────────────
service.Register()
   ├─ save user in DB
   ├─ job.Client.Enqueue(task) ──►  queue "default"  ──────►  handleWelcomeEmailTask()
   └─ return 201 ✅ (fast)          [ {to, first_name} ]           └─ email.SendWelcomeEmail()
                                                                      fails? → retried (max 3)
```

## Library: [Asynq](https://github.com/hibiken/asynq)

A Redis-backed task queue for Go. It has two halves:

- **`asynq.Client`**: *produces* tasks (enqueue)
- **`asynq.Server`**: *consumes* tasks (runs handlers with N concurrent workers)

## Files

| File             | What it contains |
|------------------|------------------|
| `job.go`         | `JobService` with `Client` (public, for enqueuing) and `server` (private). `NewJobService` sets **concurrency = 10** and **3 priority queues**. `Start()` registers handlers on a `ServeMux` and starts processing; `Stop()` shuts down. |
| `email_tasks.go` | Task **type names** (`TaskWelcome = "email:welcome"`), **payload structs** (`WelcomeEmailPayload`) and **constructors** (`NewWelcomeEmailTask`) that set `MaxRetry(3)`, `Queue("default")`, `Timeout(30s)` |
| `handlers.go`    | The **consumer** side: `InitHandlers` creates the email client; `handleWelcomeEmailTask` unmarshals the payload, sends the email and logs the outcome |

## Priority queues

```go
Queues: map[string]int{
    "critical": 6,   // processed ~60% of the time
    "default":  3,   // ~30%
    "low":      1,   // ~10%
}
```

These are **weights**, not strict priority. A flood of `critical` tasks can't
completely starve `low`.

## How to enqueue (from a service)

```go
task, err := job.NewWelcomeEmailTask(user.Email, user.FirstName)
if err != nil { return err }

if _, err := s.server.Job.Client.EnqueueContext(ctx, task); err != nil {
    return fmt.Errorf("enqueue welcome email: %w", err)
}
```

## Adding a new job type (recipe)

1. `email_tasks.go` (or a new `xxx_tasks.go`): add a `TaskXxx` const, a payload
   struct and a `NewXxxTask(...)` constructor.
2. `handlers.go`: add `func (j *JobService) handleXxxTask(ctx, t) error`.
3. `job.go` → `Start()`: `mux.HandleFunc(TaskXxx, j.handleXxxTask)`.
4. Enqueue it from a service.

## Production notes

- **Handlers must be idempotent.** A task can run more than once (crash
  after sending, before ack). Design so that "twice" is harmless.
- Return an `error` to trigger a retry. Return `nil` (or wrap with
  `asynq.SkipRetry`) when retrying will never help, e.g. bad payload.
- Here the worker runs **inside the API process**. At scale you would move
  it to its own binary (`cmd/worker`) so you can scale workers and API
  separately.
- `emailClient` is a package-level variable set by `InitHandlers`. It works, but
  storing it as a field on `JobService` is cleaner and easier to test.
- [Asynqmon](https://github.com/hibiken/asynqmon) gives you a web UI to inspect queues.
