# 07 · Configuration: the 12-factor way

## Principle: config lives in the environment

The [12-Factor App](https://12factor.net/config) rule: **anything that changes
between deploys** (DB host, API keys, ports) comes from **environment
variables**, never from code.

- The **same binary** runs on your laptop, in staging and in production.
- **Secrets never get committed** (`.env` is in `.gitignore`; only
  `.env.sample` with fake values is committed).
- Platforms (Docker, Kubernetes, Railway, Fly, ECS) all speak env vars.

## The naming scheme

```
BOILERPLATE_ DATABASE . MAX_OPEN_CONNS
└── prefix ┘ └section┘   └── key ───┘
          ↓ strip prefix, lower-case
      database.max_open_conns
          ↓ koanf "." delimiter → nested map
      { database: { max_open_conns: "25" } }
          ↓ Unmarshal with `koanf` tags (weakly typed: "25" → int 25)
      cfg.Database.MaxOpenConns == 25
```

## Validation = fail fast

```go
type ServerConfig struct {
    Port string `koanf:"port" validate:"required"`
}
```

If `BOILERPLATE_SERVER.PORT` is missing, startup prints something like:

```
FTL config validation failed error="Key: 'Config.Server.Port' Error:Field validation for 'Port' failed on the 'required' tag"
```

That is much better than discovering it when the first request arrives.

## Precedence

1. Real environment variables (from shell, Docker, k8s)
2. `.env` file (godotenv does **not** override existing vars)
3. Code defaults (only for `Observability`)

## Environments (`BOILERPLATE_PRIMARY.ENV`)

| Value | Effect in code |
|-------|----------------|
| `local` | **No auto-migration**; SQL queries logged to console |
| `development` / anything else | Auto-migrate on boot; console logs; stack traces |
| `production` | Auto-migrate; **JSON logs** (if format=json) forwarded to New Relic; no stack field |

## Secrets in production

`.env` files are fine locally. In production prefer:
- your platform's secret store (Kubernetes Secrets, AWS Secrets Manager, Doppler,
  1Password, Vault) **injected as env vars**,
- never baking secrets into Docker images,
- rotating keys and **never logging** config structs that contain secrets.

## Typed config is documentation

Reading `config.go` tells you every knob the service has. Compare that with
`os.Getenv("X")` calls scattered across 40 files.

See also [08 · Env reference](08-env-reference.md) and
[`internal/config/README.md`](../internal/config/README.md).
