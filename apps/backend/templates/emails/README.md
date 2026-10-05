# `templates/emails/` — generated HTML email templates

## Files

| File           | Used by | Placeholders |
|----------------|---------|--------------|
| `welcome.html` | `email.TemplateWelcome` → `Client.SendWelcomeEmail` | `{{.UserFirstName}}` |

## These files are GENERATED — edit the source instead

HTML email is painful to write by hand (tables, inline styles, Outlook quirks).
So the templates are authored as **React components** with
[React Email](https://react.email) and exported to static HTML:

```
packages/emails/src/templates/welcome.tsx      ← edit this
        │
        │  cd packages/emails
        │  bun run dev      → live preview at http://localhost:3001
        │  bun run export   → writes ../../apps/backend/templates/emails/*.html
        ▼
apps/backend/templates/emails/welcome.html     ← used by Go at runtime
```

In the `.tsx` file the Go placeholder is used as a literal default prop value:
`userFirstName = "{{.UserFirstName}}"`. React Email writes it into the HTML
unchanged, and Go's `html/template` fills it at send time.

## How Go uses them

```go
tmpl, _ := template.ParseFiles("templates/emails/welcome.html")
tmpl.Execute(&buf, map[string]string{"UserFirstName": "Srijit"})
```

The file name (without `.html`) **must equal** the `email.Template` constant
value (`"welcome"`).

## Adding a template (checklist)

1. `packages/emails/src/templates/my-email.tsx`
2. `bun run export` → `my-email.html` appears here
3. `internal/lib/email/templates.go` → `TemplateMyEmail Template = "my-email"`
4. `internal/lib/email/emails.go` → `SendMyEmail(...)`
5. `internal/lib/email/preview.go` → sample data
