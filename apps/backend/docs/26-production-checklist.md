# 26 · Production readiness checklist

Use this before you ship a service built on this boilerplate.

## Configuration & secrets
- [ ] `PRIMARY.ENV=production`, `LOGGING.FORMAT=json`, `LOGGING.LEVEL=info`
- [ ] Secrets come from a secret manager, not committed files
- [ ] `DATABASE.SSL_MODE=require` (or `verify-full`)
- [ ] `CORS_ALLOWED_ORIGINS` is your real frontend origin (not `*`)
- [ ] Redis address in `host:port` form

## Build & deploy
- [ ] Multi-stage Dockerfile: build with `golang:1.24`, run on a small image
      (`gcr.io/distroless/static` or `alpine`), `CGO_ENABLED=0`
- [ ] Copy `static/` and `templates/` into the image (or embed them)
- [ ] Run as a non-root user
- [ ] Image tagged with the git SHA
- [ ] CI: lint → test (with `-race`) → vulncheck → build → deploy

## Runtime
- [ ] SIGTERM handled; shutdown timeout < platform grace period
- [ ] Liveness (`/livez`) and readiness (`/status`) probes configured separately
- [ ] At least 2 replicas behind a load balancer
- [ ] DB pool sized for `replicas × max_conns < db max_connections`
- [ ] Pool settings actually applied (see gotcha #6)
- [ ] HTTP server timeouts set (already done)
- [ ] Request body size limit middleware

## Database
- [ ] Automated backups + tested restore
- [ ] Migrations are backward compatible (expand/contract)
- [ ] Indexes for all foreign keys and hot `WHERE` columns
- [ ] Least-privilege app DB user

## Observability
- [ ] New Relic (or OpenTelemetry) license configured
- [ ] Alerts: error rate, p95 latency, health check failures, queue backlog
- [ ] Logs searchable by `request_id` and `user_id`
- [ ] Dashboards for RPS, latency, errors, DB time, job throughput

## Security
- [ ] TLS everywhere; HSTS
- [ ] Rate limits appropriate (and shared across replicas if needed)
- [ ] `/docs` hidden or protected
- [ ] Trusted proxy / `IPExtractor` configured so `RealIP()` can't be spoofed
- [ ] `govulncheck` clean
- [ ] Dependency updates automated (Dependabot/Renovate)

## Jobs & email
- [ ] Job handlers idempotent
- [ ] Failed (archived) tasks monitored
- [ ] Email sender domain verified (SPF/DKIM/DMARC)
- [ ] Consider a separate worker deployment

## Docs & team
- [ ] `openapi.json` regenerated and matches the Go handlers
- [ ] README explains how to run locally in under 10 minutes
- [ ] Runbook: how to roll back, how to run a migration down, who's on call
