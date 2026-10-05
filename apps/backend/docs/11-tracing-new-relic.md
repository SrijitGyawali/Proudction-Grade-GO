# 11 · Tracing, APM and New Relic

## Logs vs metrics vs traces: the three pillars

| Pillar | Answers | Example |
|--------|---------|---------|
| **Logs** | *What happened* in detail? | "failed to send email to x@y.com: timeout" |
| **Metrics** | *How much / how often*? | p95 latency = 120 ms, error rate = 0.3% |
| **Traces** | *Where did the time go* in this request? | 3 ms handler → 95 ms SQL → 2 ms JSON |

New Relic (an APM: Application Performance Monitoring tool) collects all three.
This codebase wires it in at several points:

| Where | Integration | What you get |
|-------|-------------|--------------|
| `logger.NewLoggerService` | `newrelic.NewApplication` | the agent itself |
| `middleware.TracingMiddleware.NewRelicMiddleware` | `nrecho-v4` | one **transaction** per HTTP request (name, duration, status) |
| `middleware.TracingMiddleware.EnhanceTracing` | custom attributes | ip, user-agent, request.id, user.id, status; `NoticeError` |
| `handler.handleRequest` | custom attributes | validation/handler durations and status |
| `database.New` | `nrpgx5` | every SQL query as a **datastore segment** |
| `server.New` | `nrredis-v9` | every Redis command as a segment |
| `logger.NewLoggerWithService` | `zerologWriter` | logs forwarded and **linked to traces** |
| `logger.WithTraceContext` | trace/span ids | `trace.id` in every request log line |
| health / rate limit | `RecordCustomEvent` | `HealthCheckError`, `RateLimitHit` events you can chart and alert on |
| `nrpkgerrors.Wrap` | error stack traces | errors grouped with stack in the Errors inbox |

## Optional by design

Every integration checks `GetApplication() != nil`. **Without a license key the
app runs exactly the same**, just without telemetry. Library code should not
assume a vendor is present.

## Distributed tracing

With `DISTRIBUTED_TRACING_ENABLED=true`, New Relic propagates trace headers
(`traceparent`, `newrelic`) across services. If your frontend or another
service also uses New Relic, you see one trace spanning all of them.

## Using it in your own code

```go
txn := newrelic.FromContext(ctx)          // nil-safe: methods on nil txn are no-ops
defer txn.StartSegment("todo.ImportCSV").End()

txn.AddAttribute("import.rows", len(rows))
txn.NoticeError(err)
```

For a background job, there is no HTTP transaction. Start one yourself:

```go
txn := nrApp.StartTransaction("job:email:welcome")
defer txn.End()
ctx = newrelic.NewContext(ctx, txn)
```

## Alternatives

The same ideas apply with **OpenTelemetry** (vendor-neutral standard) + Grafana /
Datadog / Honeycomb / Jaeger. Many teams start with a vendor agent (like here)
and move to OpenTelemetry later.

## What to alert on (starter set)

- Error rate > 1% for 5 minutes
- p95 latency > your SLO (e.g. 500 ms)
- `HealthCheckError` events > 0
- `RateLimitHit` spikes (possible abuse)
- Asynq queue size growing (jobs not keeping up)

---

Next: [12 · Database & pooling](12-database-pooling.md)
