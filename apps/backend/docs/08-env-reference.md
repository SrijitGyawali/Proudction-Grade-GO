# 08 · Environment variable reference

All variables are read from the environment or `apps/backend/.env` (copy
`.env.sample`). ✅ = required (startup fails without it).

## Primary

| Variable | Req | Example | Used for |
|----------|-----|---------|----------|
| `BOILERPLATE_PRIMARY.ENV` | ✅ | `local` | Environment name; `local` disables auto-migration and enables SQL logs |

## Server

| Variable | Req | Example | Used for |
|----------|-----|---------|----------|
| `BOILERPLATE_SERVER.PORT` | ✅ | `8080` | HTTP listen port |
| `BOILERPLATE_SERVER.READ_TIMEOUT` | ✅ | `30` | Seconds to read full request |
| `BOILERPLATE_SERVER.WRITE_TIMEOUT` | ✅ | `30` | Seconds to write response |
| `BOILERPLATE_SERVER.IDLE_TIMEOUT` | ✅ | `60` | Seconds to keep idle keep-alive connections |
| `BOILERPLATE_SERVER.CORS_ALLOWED_ORIGINS` | ✅ | `http://localhost:3000` | Allowed browser origin (**one value**; commas are not split, see [25](25-gotchas.md)) |

## Database

| Variable | Req | Example | Used for |
|----------|-----|---------|----------|
| `BOILERPLATE_DATABASE.HOST` | ✅ | `localhost` | Postgres host |
| `BOILERPLATE_DATABASE.PORT` | ✅ | `5432` | Postgres port |
| `BOILERPLATE_DATABASE.USER` | ✅ | `postgres` | DB user |
| `BOILERPLATE_DATABASE.PASSWORD` | | `secret` | DB password (URL-escaped automatically) |
| `BOILERPLATE_DATABASE.NAME` | ✅ | `boilerplate` | Database name |
| `BOILERPLATE_DATABASE.SSL_MODE` | ✅ | `disable` / `require` | TLS to DB. Use `require` (or stricter) in prod |
| `BOILERPLATE_DATABASE.MAX_OPEN_CONNS` | ✅ | `25` | Declared, **not applied** to pgxpool yet |
| `BOILERPLATE_DATABASE.MAX_IDLE_CONNS` | ✅ | `25` | Declared, not applied yet |
| `BOILERPLATE_DATABASE.CONN_MAX_LIFETIME` | ✅ | `300` | Declared, not applied yet |
| `BOILERPLATE_DATABASE.CONN_MAX_IDLE_TIME` | ✅ | `300` | Declared, not applied yet |

## Auth / integrations / Redis

| Variable | Req | Example | Used for |
|----------|-----|---------|----------|
| `BOILERPLATE_AUTH.SECRET_KEY` | ✅ | `sk_test_…` | Clerk secret key (`clerk.SetKey`) |
| `BOILERPLATE_INTEGRATION.RESEND_API_KEY` | ✅ | `re_…` | Resend email API key |
| `BOILERPLATE_REDIS.ADDRESS` | ✅ | `localhost:6379` | Redis for Asynq + health check. Use **`host:port`**, not `redis://…` |

## Observability (all optional; defaults shown)

| Variable | Default | Notes |
|----------|---------|-------|
| `BOILERPLATE_OBSERVABILITY.SERVICE_NAME` | `boilerplate` | Overwritten to `boilerplate` in code anyway |
| `BOILERPLATE_OBSERVABILITY.ENVIRONMENT` | `development` | Overwritten from `PRIMARY.ENV` in code |
| `BOILERPLATE_OBSERVABILITY.LOGGING.LEVEL` | `info` | `debug` \| `info` \| `warn` \| `error` |
| `BOILERPLATE_OBSERVABILITY.LOGGING.FORMAT` | `json` | `json` only takes effect when env is `production` |
| `BOILERPLATE_OBSERVABILITY.LOGGING.SLOW_QUERY_THRESHOLD` | `100ms` | Declared, not used yet |
| `BOILERPLATE_OBSERVABILITY.NEW_RELIC.LICENSE_KEY` | empty | Empty = New Relic disabled |
| `BOILERPLATE_OBSERVABILITY.NEW_RELIC.APP_LOG_FORWARDING_ENABLED` | `true` | |
| `BOILERPLATE_OBSERVABILITY.NEW_RELIC.DISTRIBUTED_TRACING_ENABLED` | `true` | |
| `BOILERPLATE_OBSERVABILITY.NEW_RELIC.DEBUG_LOGGING` | `false` | Agent debug logs to stdout |
| `BOILERPLATE_OBSERVABILITY.HEALTH_CHECKS.ENABLED` | `true` | Declared, not used by `/status` yet |
| `BOILERPLATE_OBSERVABILITY.HEALTH_CHECKS.INTERVAL` | `30s` | Must be ≥ 1s if the section is set |
| `BOILERPLATE_OBSERVABILITY.HEALTH_CHECKS.TIMEOUT` | `5s` | Must be ≥ 1s if the section is set |
| `BOILERPLATE_OBSERVABILITY.HEALTH_CHECKS.CHECKS` | `database,redis` | Declared, not used yet |

> ⚠️ If you set **any** `OBSERVABILITY.*` variable, the defaults are **not**
> merged in, because the struct is created from env only. Set the whole section
> (as `.env.sample` does) or none of it. Once the section exists, its struct tags
> are validated too: `LOGGING.LEVEL`, `LOGGING.FORMAT` and
> `NEW_RELIC.LICENSE_KEY` become **required**, and the health-check
> `INTERVAL`/`TIMEOUT` must be ≥ 1s. That is why `.env.sample` has a placeholder
> license key. A fake key simply makes New Relic fail to start, and the app
> keeps running without it.

## Taskfile-only

| Variable | Used by |
|----------|---------|
| `BOILERPLATE_DB_DSN` | `task migrations:up`, e.g. `postgres://postgres:pass@localhost:5432/boilerplate?sslmode=disable` |

## Minimal local `.env`

```dotenv
BOILERPLATE_PRIMARY.ENV="local"
BOILERPLATE_SERVER.PORT="8080"
BOILERPLATE_SERVER.READ_TIMEOUT="30"
BOILERPLATE_SERVER.WRITE_TIMEOUT="30"
BOILERPLATE_SERVER.IDLE_TIMEOUT="60"
BOILERPLATE_SERVER.CORS_ALLOWED_ORIGINS="http://localhost:3000"
BOILERPLATE_DATABASE.HOST="localhost"
BOILERPLATE_DATABASE.PORT="5432"
BOILERPLATE_DATABASE.USER="postgres"
BOILERPLATE_DATABASE.PASSWORD="postgres"
BOILERPLATE_DATABASE.NAME="boilerplate"
BOILERPLATE_DATABASE.SSL_MODE="disable"
BOILERPLATE_DATABASE.MAX_OPEN_CONNS="25"
BOILERPLATE_DATABASE.MAX_IDLE_CONNS="25"
BOILERPLATE_DATABASE.CONN_MAX_LIFETIME="300"
BOILERPLATE_DATABASE.CONN_MAX_IDLE_TIME="300"
BOILERPLATE_AUTH.SECRET_KEY="sk_test_xxx"
BOILERPLATE_INTEGRATION.RESEND_API_KEY="re_xxx"
BOILERPLATE_REDIS.ADDRESS="localhost:6379"
```
