# 17 · Security: what's built in and what to add

## Built in today

| Protection | Where | Details |
|------------|-------|---------|
| **Rate limiting** | `router.go` | 20 req/s per IP (token bucket, memory store) → 429 |
| **CORS** | `middleware/global.go` | Only origins in `CORS_ALLOWED_ORIGINS` can call from a browser |
| **Security headers** | `middleware.Secure()` | `X-XSS-Protection: 1; mode=block`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN` |
| **Authentication** | `middleware/auth.go` | Clerk JWT verification on protected routes |
| **Input validation** | `validation/` | Every request bound + validated before use |
| **SQL injection** | pgx | Parameterised queries (`$1`, `@name`) |
| **XSS in emails** | `html/template` | Auto-escaping |
| **Panic safety** | `middleware.Recover()` | A crash in one request doesn't kill the server |
| **Error hygiene** | `GlobalErrorHandler` | 500s never leak internals |
| **Slowloris / hung connections** | `http.Server` timeouts | Read/Write/Idle timeouts |
| **Secrets out of code** | `config` + `.gitignore` | `.env` ignored, `.env.sample` committed |
| **Request tracing** | `X-Request-ID` | Every request auditable |

## Things to add for real production

| Item | Why | How |
|------|-----|-----|
| **HTTPS / HSTS** | Encrypt traffic, force HTTPS | TLS at the load balancer; `SecureWithConfig{HSTSMaxAge: 31536000}` |
| **Content-Security-Policy** | Strong XSS defence for any HTML you serve (`/docs`) | `SecureWithConfig{ContentSecurityPolicy: "..."}` |
| **Body size limit** | Stop huge payload DoS | `echoMiddleware.BodyLimit("1M")` |
| **Request timeout** | Stop slow handlers holding resources | context-timeout middleware (see [05](05-middleware-order.md)) |
| **Distributed rate limit** | Limits that hold across replicas | Redis-based limiter |
| **Per-user rate limits** | Fairness, abuse control | identifier = user id instead of IP |
| **Trusted proxy config** | `c.RealIP()` trusts `X-Forwarded-For`; spoofable without config | `e.IPExtractor = echo.ExtractIPFromXFFHeader(echo.TrustIPRange(...))` |
| **Hide `/docs` in prod** | Don't advertise the attack surface | register route only when `env != "production"` or add auth |
| **Dependency scanning** | Known CVEs in modules | `govulncheck ./...`, Dependabot |
| **Static analysis** | Insecure patterns | `gosec` (already enabled in `.golangci.yml`) |
| **Least-privilege DB user** | Limit damage if creds leak | app user without `DROP`/superuser; separate migration user |
| **Audit logging** | Who changed what | log security-relevant actions with user id |

## OWASP Top 10 mapping (short)

| Risk | Mitigation here |
|------|-----------------|
| Broken access control | `RequireAuth` + ownership checks in services/SQL |
| Cryptographic failures | TLS, `SSL_MODE=require` for DB, Clerk handles passwords |
| Injection | pgx parameters, `html/template` |
| Insecure design | layered architecture, validation everywhere |
| Security misconfiguration | typed + validated config, fail fast |
| Vulnerable components | `govulncheck`, keep `go.mod` updated |
| Auth failures | delegated to Clerk |
| Data integrity failures | DB constraints, migrations in VCS |
| Logging & monitoring failures | structured logs, New Relic, request ids |
| SSRF | validate any user-supplied URL before the server fetches it |

---

Next: [18 · Health checks](18-health-checks.md)
