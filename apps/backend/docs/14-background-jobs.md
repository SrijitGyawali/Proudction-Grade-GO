# 14 · Background jobs: thinking asynchronously

## The rule of thumb

> If the user doesn't need the result **in this response**, and the work is
> **slow or unreliable**, do it in a background job.

| Sync (in request) | Async (job) |
|-------------------|-------------|
| Validate input | Send email / SMS / push |
| Write the main DB row | Generate thumbnails / PDFs |
| Return the created resource | Call slow 3rd-party APIs, webhooks |
| | Recalculate stats, cleanup, sync to search index |

## Why not just `go sendEmail()`?

A bare goroutine is **fire-and-forget**:
- if the process restarts (deploy!), the work is **lost**,
- no retries, no backoff, no visibility,
- no limit on how many run at once.

A **queue** (Redis + Asynq) persists the task, retries with exponential backoff,
limits concurrency, and lets you inspect what's pending or failed.

## Lifecycle of a task in Asynq

```
Enqueue ──► pending ──► active ──► completed ✅
                          │
                          └─ error ──► retry (backoff: ~seconds → minutes) ──► active …
                                         │
                                         └─ MaxRetry reached ──► archived ❌ (dead letter)
```

## Key properties to design for

1. **Idempotency.** A task can run **more than once** (e.g. worker crashes after
   sending, before acknowledging). Make re-runs harmless: check "already sent?",
   use unique keys, or use provider idempotency keys.
2. **Small payloads.** Put **IDs** in the payload, not whole objects, and load
   fresh data in the handler. The data may have changed since enqueue.
3. **Timeouts.** `asynq.Timeout(30*time.Second)` stops a stuck handler.
4. **Uniqueness.** `asynq.Unique(time.Hour)` prevents duplicate tasks (e.g. two
   welcome emails from a double-click).
5. **Scheduling.** `asynq.ProcessIn(10*time.Minute)` or `asynq.ProcessAt(t)`
   for "send reminder later". `asynq.Scheduler` covers cron-like recurring jobs.

## Example: welcome email on sign-up

```go
// service/user.go
func (s *UserService) OnSignUp(ctx context.Context, u *model.User) error {
    task, err := job.NewWelcomeEmailTask(u.Email, u.FirstName)
    if err != nil {
        return err
    }
    _, err = s.server.Job.Client.EnqueueContext(ctx, task,
        asynq.Unique(24*time.Hour))   // never two welcome emails within a day
    return err
}
```

## Scaling the worker separately

Today the worker runs inside the API process (`server.New` calls
`jobService.Start()`). To scale independently:

```
cmd/go-boilerplate/   API only  (don't call Job.Start, only use Job.Client)
cmd/worker/           Worker only (no HTTP server, just jobService.Start + wait for signal)
```

Then you can run 2 API pods and 10 worker pods, or the other way round.

## Transactional outbox (advanced)

Problem: you insert a user **and** enqueue an email. If the DB commit succeeds
but enqueue fails (Redis down), the email is lost. If you enqueue first and the
DB commit fails, you email a user that doesn't exist.

Solution: write an `outbox` row **in the same DB transaction**, and have a
poller move outbox rows into the queue. Look this up when correctness matters
(payments, orders).

---

Next: [15 · Email pipeline](15-email-pipeline.md)
