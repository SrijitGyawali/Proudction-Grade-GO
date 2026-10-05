# 18 · Health checks

## Endpoint

```
GET /status
```

```json
{
  "status": "healthy",
  "timestamp": "2026-10-05T14:00:00Z",
  "environment": "production",
  "checks": {
    "database": { "status": "healthy", "response_time": "1.2ms" },
    "redis":    { "status": "healthy", "response_time": "0.4ms" }
  }
}
```

| Situation | HTTP | `status` |
|-----------|------|----------|
| DB ok, Redis ok | 200 | healthy |
| DB ok, Redis down | **200** | healthy (`checks.redis.status = unhealthy`) |
| DB down | **503** | unhealthy |

## Who calls it?

- **Load balancers** (AWS ALB, GCP LB, Nginx): stop routing to an instance that
  returns non-2xx.
- **Kubernetes probes**: restart or stop routing to a pod.
- **Uptime monitors** (Better Uptime, Pingdom, New Relic Synthetics): page you.

## Liveness vs readiness: an important distinction

| Probe | Question | Should check dependencies? | On failure |
|-------|----------|----------------------------|-----------|
| **Liveness** | Is the process stuck/dead? | **No**, only "can I respond?" | container is **restarted** |
| **Readiness** | Can I serve traffic right now? | **Yes** (DB, etc.) | removed from load balancer, **not** restarted |

`/status` here is a **readiness**-style check. If you used it for liveness and
the DB went down, Kubernetes would restart **every pod** in a loop, which only
makes the outage worse. Add a trivial liveness endpoint:

```go
r.GET("/livez", func(c echo.Context) error { return c.NoContent(http.StatusOK) })
```

```yaml
livenessProbe:  { httpGet: { path: /livez,  port: 8080 }, periodSeconds: 10 }
readinessProbe: { httpGet: { path: /status, port: 8080 }, periodSeconds: 10 }
```

## Design notes

- Each check has its **own 5 s timeout**, so a hung dependency can't hang the
  health check forever.
- Failures emit a `HealthCheckError` New Relic event, which you can alert on.
- Health endpoints should be **cheap**: a `Ping`, not a heavy query.
- They should be **public**, with no auth, but they reveal some info
  (environment, error messages). Consider trimming the body in production.

---

Next: [19 · Graceful shutdown](19-graceful-shutdown.md)
