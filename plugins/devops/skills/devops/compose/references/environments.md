# Dev vs prod: overrides, profiles, env, secrets

## Override files

`docker compose up` automatically merges `compose.yaml` + `compose.override.yaml`
if the second exists. That is the whole mechanism: the base file holds what is
true everywhere, the override holds development conveniences.

```yaml
# compose.yaml — shared
services:
  api:
    build: { context: ., target: runtime }
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
    restart: unless-stopped
```

```yaml
# compose.override.yaml — development, applied by default
services:
  api:
    build: { target: dev }
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--reload"]
    volumes: ["./app:/app/app"]
    environment:
      LOG_LEVEL: debug
    ports: ["8000:8000"]
```

Production then ignores the override explicitly:

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

Merge semantics matter: scalars are replaced, **lists are appended**. Two
override files both adding `ports:` gives you both port mappings, not the last
one. To replace a list, use `!reset` (`ports: !reset []`) or `!override`.

## Profiles

Optional services that shouldn't start with the default `up`:

```yaml
  adminer:
    image: adminer:5
    profiles: [tools]
    ports: ["8080:8080"]
  seed:
    build: .
    command: ["./seed"]
    profiles: [seed]
```

`docker compose up` skips them; `docker compose --profile tools up -d` includes
them. Also enabled by `COMPOSE_PROFILES=tools` in `.env`. Good for admin UIs,
seeders, load generators — anything you want defined but not running.

A service with a profile that another service `depends_on` is started
automatically, so a profiled dependency won't silently break the graph.

## Where environment variables come from

Two different mechanisms, often confused:

**Interpolation into the compose file itself** — only from the `.env` file next
to the compose file, plus the shell environment:

```yaml
    image: postgres:${POSTGRES_VERSION:-17}-alpine
```

`${VAR:-default}` substitutes when unset *or empty*, `${VAR-default}` only when
unset, `${VAR:?message}` fails the command with that message. Use the last one
for things that must be provided.

**Variables inside the container** — `environment:` (wins) then `env_file:`.
A shell variable beats `environment:` only when written as `- VAR` with no value,
which passes it through from the host.

```yaml
    env_file: [.env.local]
    environment:
      LOG_LEVEL: debug        # beats anything in .env.local
      HOME_DIR:               # no value: take it from the host shell
```

Precedence, highest first: shell (for pass-through keys) → `environment:` →
`env_file:` → image `ENV`. `docker compose config` prints the resolved result;
when a variable is not what you expect, that command answers it in one step.

Keep `.env` out of git and commit `.env.example`. `.env` is for local values and
for interpolation, not a secret store — it is read in plaintext by anything on
the machine.

## Secrets

```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password
    secrets: [db-password]
secrets:
  db-password:
    file: ./secrets/db-password.txt     # gitignored
```

Mounted at `/run/secrets/<name>`, not present in `docker inspect` or in the
image. Most official images accept the `_FILE` suffix for exactly this. Under
Swarm, `external: true` reads from the cluster's secret store instead.

## Resource limits

Without `deploy.resources`, one container can take the whole machine — a Java or
Node process reads the *host's* memory and sizes its heap accordingly.

```yaml
    deploy:
      resources:
        limits:   { cpus: "1.0", memory: 512M }
        reservations: { cpus: "0.25", memory: 256M }
```

`deploy.resources.limits` is honoured by `docker compose up` (the rest of
`deploy:` is Swarm-only and ignored). A container exceeding the memory limit is
killed with exit code 137 — see `devops:container-debugging`.

## Sanity commands

```bash
docker compose config                      # resolved, merged, validated
docker compose config --services           # what would run
docker compose --profile tools config      # with a profile enabled
docker compose -f compose.yaml -f compose.prod.yaml config   # what prod really is
```

Run the last one before deploying. It is the only way to see the actual merged
result rather than the file you think you wrote.
