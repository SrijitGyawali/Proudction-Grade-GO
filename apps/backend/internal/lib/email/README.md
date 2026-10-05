# `internal/lib/email/` — transactional emails via Resend

## Job of this package

Render an HTML template with some data and send it with the
[Resend](https://resend.com) API.

## Files

| File           | What it contains |
|----------------|------------------|
| `client.go`    | `Client` (wraps `*resend.Client` + logger), `NewClient(cfg, logger)`, and the generic `SendEmail(to, subject, template, data)` |
| `templates.go` | `type Template string` and the list of template names (`TemplateWelcome = "welcome"`). It works like an **enum**, so a typo becomes a compile error. |
| `emails.go`    | One **friendly method per email**: `SendWelcomeEmail(to, firstName)`. It builds the data map and picks the subject + template. |
| `preview.go`   | `PreviewData`: sample values for each template, for previews and tests |

## Flow of one email

```
SendWelcomeEmail("a@b.com", "Srijit")
   │  data = { "UserFirstName": "Srijit" }
   ▼
SendEmail(to, "Welcome to Boilerplate!", TemplateWelcome, data)
   │  1. path = "templates/emails/welcome.html"
   │  2. html/template.ParseFiles(path)
   │  3. tmpl.Execute(&buf, data)      ← fills {{.UserFirstName}}
   │  4. resend.Emails.Send({From, To, Subject, Html})
   ▼
error or nil
```

## Where do the HTML templates come from?

They are **generated**, not hand-written:

```
packages/emails/src/templates/welcome.tsx   (React Email component)
          │  bun run export   (in packages/emails)
          ▼
apps/backend/templates/emails/welcome.html  (plain HTML with {{.UserFirstName}})
```

See [`../../../templates/emails/README.md`](../../../templates/emails/README.md).

## Why `html/template` and not `text/template`?

`html/template` **auto-escapes** values. If a user's first name is
`<script>alert(1)</script>`, it is rendered as harmless text, not code.

## Emails should normally be sent from a background job

Sending an email is a slow network call (100 ms – several seconds) that can
fail. Do **not** make the user's HTTP request wait for it. The intended path is:

```
service → job.NewWelcomeEmailTask(...) → Asynq (Redis) → worker → email.SendWelcomeEmail(...)
```

See [`../job/README.md`](../job/README.md).

## Adding a new email (recipe)

1. Create `packages/emails/src/templates/password-reset.tsx`, then export to HTML.
2. `templates.go`: add `TemplatePasswordReset Template = "password-reset"`.
3. `emails.go`: add `func (c *Client) SendPasswordResetEmail(to, link string) error`.
4. `preview.go`: add sample data.
5. Add a job task for it in `lib/job/`.

## Notes / improvements

- The `From` address is hard-coded to `Boilerplate <onboarding@resend.dev>`
  (Resend's sandbox sender). For production, move it into config and verify your
  own domain in Resend.
- Templates are parsed from disk **on every send**, relative to the working
  directory. You could parse them once at startup, or `//go:embed` them.
