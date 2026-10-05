# `internal/lib/` — reusable building blocks

## What goes in `lib/`?

Code that is **useful to several layers** but is **not business logic** and
**not HTTP-specific**. Think of each sub-folder as a small internal library
that wraps an external service or a technical concern.

```
lib/
├── email/   wraps the Resend API + HTML templates          → "send this email"
├── job/     wraps Asynq (Redis task queue)                 → "do this later, in the background"
└── utils/   tiny generic helpers                           → "pretty print JSON"
```

## How to decide if code belongs here

| Question | If yes… |
|----------|---------|
| Does it encode a rule of *your product* ("a user can have at most 5 projects")? | → `service/` |
| Does it run SQL? | → `repository/` |
| Does it read `echo.Context`? | → `handler/` or `middleware/` |
| Is it a **client for a third-party system** (email, SMS, S3, Stripe, OpenAI…)? | → `lib/<name>/` ✅ |
| Is it a small pure helper with no dependencies? | → `lib/utils/` ✅ |

## Why wrap third-party SDKs at all?

```go
// ❌ service calls Resend SDK directly
resend.NewClient(key).Emails.Send(&resend.SendEmailRequest{...})

// ✅ service calls our wrapper
emailClient.SendWelcomeEmail(to, firstName)
```

1. **One place to change** if you switch providers (Resend → SES).
2. **Domain-friendly API**: `SendWelcomeEmail` instead of raw request structs.
3. **Testable**: you can put an interface in front of the wrapper and fake it in tests.
4. **Consistent logging and error wrapping** around every external call.

## Future examples you might add

- `lib/storage/`: S3 / R2 uploads
- `lib/cache/`: typed helpers over Redis
- `lib/payments/`: Stripe client
- `lib/ai/`: an LLM client
