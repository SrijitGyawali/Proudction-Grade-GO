# `internal/validation/` — bind & validate incoming requests

## Job of this package

Take the raw HTTP request, **bind** it into a Go struct (JSON body, path params,
query params), **validate** it, and on failure return a `400` with
**per-field error messages** the frontend can show next to inputs.

## Files

| File       | What it contains |
|------------|------------------|
| `utils.go` | The `Validatable` interface, `BindAndValidate(c, payload)`, conversion of validator errors into `[]errs.FieldError` with human messages, `CustomValidationError(s)` for hand-written rules, and `IsValidUUID` |

## The contract: `Validatable`

```go
type Validatable interface {
    Validate() error
}
```

Every request struct used with `handler.Handle(...)` must implement it. This
puts **the rules next to the data**:

```go
type CreateTodoRequest struct {
    Title    string `json:"title"    validate:"required,min=1,max=200"`
    Priority string `json:"priority" validate:"omitempty,oneof=low medium high"`
}

func (r *CreateTodoRequest) Validate() error {
    return validator.New().Struct(r)
}
```

## Binding sources (Echo's `c.Bind`)

| Tag     | Read from |
|---------|-----------|
| `json`  | request body |
| `param` | path, e.g. `/todos/:id` → `ID string \`param:"id"\`` |
| `query` | `?page=2` → `Page int \`query:"page"\`` |

## Messages produced

| Validator tag | Message |
|---------------|---------|
| `required`    | `is required` |
| `min` / `max` | `must be at least N characters` / `must not exceed N characters` (strings) or `must be at least N` (numbers) |
| `oneof`       | `must be one of: a b c` |
| `email`       | `must be a valid email address` |
| `e164`        | `must be a valid phone number with country code` |
| `uuid`        | `must be a valid UUID` |
| `dive`        | `some items are invalid` |
| other         | `field: tag:param` |

Response:

```json
{ "code": "BAD_REQUEST", "message": "Validation failed", "status": 400, "override": true,
  "errors": [ { "field": "title", "error": "is required" } ] }
```

## Custom rules (cross-field, business-ish)

```go
func (r *DateRange) Validate() error {
    if err := validator.New().Struct(r); err != nil {
        return err
    }
    if r.End.Before(r.Start) {
        return validation.CustomValidationErrors{{Field: "end", Message: "must be after start"}}
    }
    return nil
}
```

## Gotchas

- `extractValidationErrors` uses a type assertion `err.(CustomValidationErrors)`
  **without** the `ok` check. If `Validate()` returns any *other* error type, the
  server **panics**, and `Recover` turns that into a 500. Only return
  `validator.ValidationErrors` or `CustomValidationErrors` from `Validate()`.
- Bind errors are parsed by splitting `err.Error()` on `,` and `message=`. If
  Echo ever changes that text format, this would panic too. A safer version uses
  `errors.As(err, &echoHTTPErr)`.
- `validator.New()` caches struct info **per instance**. Creating one per
  request is wasteful. Use one package-level `var validate = validator.New()`.
- Field names come from the **Go field name**, lower-cased (`FirstName` →
  `firstname`), not from the json tag. Use `validate.RegisterTagNameFunc` to
  report `firstName` instead.
