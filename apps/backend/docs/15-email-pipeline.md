# 15 · The email pipeline, end to end

```
 DESIGN TIME                                         RUN TIME
 ───────────                                         ────────
 packages/emails/src/templates/welcome.tsx           service: enqueue job.NewWelcomeEmailTask(to, name)
        │  React Email (JSX + Tailwind)                     │
        │  bun run dev    → preview :3001                   ▼
        │  bun run export                            Redis queue "default"
        ▼                                                   │
 apps/backend/templates/emails/welcome.html                 ▼
   (static HTML with {{.UserFirstName}})            job.handleWelcomeEmailTask
        │                                                   │
        └───────────── read at send time ───────────► email.SendWelcomeEmail
                                                            │ html/template fill
                                                            ▼
                                                     Resend API ──► inbox
```

## Why this split?

- **Designers and frontend devs** work in React with live preview. No Go
  needed.
- **Go** only fills in variables. It doesn't care how the HTML was made.
- **Sending is async**, so a slow or failing email provider never slows down
  or breaks an API request.

## Layers inside the backend

| Layer | File | Responsibility |
|-------|------|----------------|
| Template names | `lib/email/templates.go` | typed constants, so a typo fails at compile time |
| Generic sender | `lib/email/client.go` `SendEmail` | load file → render → call Resend |
| Specific emails | `lib/email/emails.go` | `SendWelcomeEmail(to, firstName)`: subject + data map |
| Preview data | `lib/email/preview.go` | sample values |
| Task definition | `lib/job/email_tasks.go` | task type + JSON payload + retry/timeout options |
| Task handler | `lib/job/handlers.go` | decode payload → call `SendWelcomeEmail` |

## Production considerations

- **Sender domain**: verify your domain in Resend and configure SPF, DKIM and
  DMARC DNS records, or your emails will land in spam. Move the `From` address
  into config.
- **Bounces & complaints**: handle Resend webhooks; stop emailing addresses that
  bounce.
- **Unsubscribe**: marketing emails legally need an unsubscribe link.
  Transactional ones (password reset) don't.
- **Testing**: in dev, use Resend's test addresses or a local catcher like
  Mailpit, and never email real users from staging.
- **Rate limits**: providers limit sends per second. The queue's concurrency
  setting helps smooth bursts.
- **Localisation**: one template per language, or pass translated strings in
  the data map.

---

Next: [16 · Authentication with Clerk](16-authentication-clerk.md)
