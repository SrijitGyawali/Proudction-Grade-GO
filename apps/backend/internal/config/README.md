# `internal/config/` — typed configuration from environment variables

## Job of this package

Turn a pile of **strings in environment variables** into **one validated, typed
Go struct** (`*config.Config`) that the rest of the app can trust.

If config is wrong, the app should **crash at startup** with a clear message.
It should not fail at 3 a.m. on the first request that touches a missing value.

## Files

| File               | What it contains |
|--------------------|------------------|
| `config.go`        | The root `Config` struct, sub-structs (`ServerConfig`, `DatabaseConfig`, `RedisConfig`, `AuthConfig`, `IntegrationConfig`, `Primary`) and `LoadConfig()` |
| `observability.go` | `ObservabilityConfig` (logging, New Relic, health checks), sensible defaults (`DefaultObservabilityConfig`) and a custom `Validate()` |

## How loading works (`LoadConfig`)

```
.env file ──(godotenv/autoload)──► process env vars
                                        │
         BOILERPLATE_DATABASE.HOST=...  │  koanf env provider:
                                        │   1. keep only vars starting with BOILERPLATE_
                                        │   2. strip the prefix
                                        │   3. lower-case → "database.host"
                                        ▼
                         koanf key tree  { database: { host: ... } }
                                        │  k.Unmarshal("", &Config{})  (uses `koanf:"..."` tags)
                                        ▼
                               *Config struct
                                        │  validator.Struct()  (uses `validate:"required"` tags)
                                        ▼
                      Observability defaults + Validate()
                                        ▼
                                 returned to main()
```

### Key libraries

- **`github.com/joho/godotenv/autoload`**: the blank import `_ "…/autoload"`
  runs an `init()` that reads `.env` from the **current working directory**
  into the process environment. Real env vars win over `.env`.
- **`github.com/knadh/koanf`**: a lightweight config library. The `.` in
  `koanf.New(".")` is the **key delimiter**, which is why env names contain dots:
  `BOILERPLATE_SERVER.PORT` → `server.port` → `Config.Server.Port`.
- **`github.com/go-playground/validator`**: reads the `validate:"required"` struct
  tags. A missing required value makes the app exit at startup.

## Struct tags explained

```go
Port string `koanf:"port" validate:"required"`
//            ▲ key name in koanf   ▲ rule checked by validator
```

## Design decisions worth learning

1. **One prefix (`BOILERPLATE_`)** keeps your app's vars apart from the rest of
   the environment, such as `PATH` and `HOME`.
2. **Nested structs mirror nested keys.** Each team or sub-system gets its own
   struct, so a function can receive only `cfg.Database` instead of everything.
3. **Fail fast.** `logger.Fatal()` exits the process on any config error.
4. **Defaults for optional sections.** `Observability` is a pointer. If nothing
   is set, `DefaultObservabilityConfig()` fills it in.
5. **Some values are forced.** `ServiceName` is always `"boilerplate"`, and
   `Observability.Environment` is always copied from `Primary.Env`. Setting
   `BOILERPLATE_OBSERVABILITY.ENVIRONMENT` has no effect.

## Gotchas

- **Dots in env var names.** `.env` files and Docker/Kubernetes allow them, but
  a normal shell `export BOILERPLATE_SERVER.PORT=8080` does **not**. Use the `.env`
  file, `env "BOILERPLATE_SERVER.PORT=8080" go run …`, or your platform's
  env settings.
- **Lists are not split on commas.** koanf's default decoder converts a string
  into a one-element slice. `CORS_ALLOWED_ORIGINS="http://a.com,http://b.com"`
  becomes **one** origin string `"http://a.com,http://b.com"`. One origin works
  fine. For several you would add `mapstructure.StringToSliceHookFunc(",")`
  through `k.UnmarshalWithConf`.
- **Durations** like `"30s"` / `"100ms"` *are* parsed (koanf includes the
  duration hook), which is why `HEALTH_CHECKS.INTERVAL="30s"` works.
- `.env` is read relative to the **working directory**. Run the app from
  `apps/backend`.

## Adding a new setting (recipe)

1. Add a field to the right struct with `koanf:"..."` and `validate:"..."` tags.
2. Add it to `.env.sample` (and your `.env`).
3. Read it where needed: `s.Config.MyArea.MyField`.

See also: [`../../docs/07-configuration.md`](../../docs/07-configuration.md) and
[`../../docs/08-env-reference.md`](../../docs/08-env-reference.md).
