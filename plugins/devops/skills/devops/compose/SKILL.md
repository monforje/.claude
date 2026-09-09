---
name: compose
description: Write and fix Docker Compose stacks — services, networks, named volumes, healthchecks and startup ordering, dev/prod overrides and profiles, environment variables and secrets, plus ready blocks for Postgres, MongoDB, Redis, RabbitMQ, Kafka, NATS and for Go, FastAPI and React+nginx applications. Use whenever a compose.yaml is being written or changed, a database or broker is being added to a stack, services start in the wrong order or race each other, a container can't reach another by name, or a stack needs separating into development and production. Not for a single image's Dockerfile (use the dockerfile skill), not for Traefik routing and TLS (use the traefik skill).
---

# Docker Compose

Compose describes one machine's worth of containers. Most of what goes wrong is
startup ordering, name resolution, or data that vanished with a volume.

## The shape

```yaml
name: myapp                      # project name; otherwise the directory name

services:
  api:
    build:
      context: .
      target: runtime            # which Dockerfile stage
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy
    ports: ["8000:8000"]
    restart: unless-stopped

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes: [db-data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s

volumes:
  db-data:
```

**No `version:` key.** It was removed from the Compose Specification; leaving it
in gets you a warning and nothing else. Any example that starts with
`version: '3.8'` predates Compose v2.

## The four things that cause most bugs

**1. `depends_on` without `condition` means nothing useful.** Plain `depends_on:
[db]` waits for the container to be *created*, not for Postgres to accept
connections. The app starts, fails to connect, and exits. Always pair it with a
`healthcheck` on the dependency and `condition: service_healthy`. Per-service
healthcheck commands: `references/datastores.md`, `references/brokers.md`.

Use `condition: service_completed_successfully` for one-shot jobs (migrations).

**2. Services reach each other by service name, on the container port.** From
`api`, Postgres is `db:5432` — always 5432, even if you published it as
`15432:5432`. `localhost` inside a container is that container. `ports:` is only
for reaching a service *from the host*; services that only talk to each other
don't need it, and publishing a database to `0.0.0.0` is how databases get found
by scanners.

**3. Named volumes are the data; bind mounts are the source.** `db-data:/var/lib/
postgresql/data` survives `down` and `up`. It does **not** survive `down -v` —
that is the command that deletes your database. A bind mount (`./src:/app/src`)
is for live-reloading code in development, not for data.

**4. Env vars come from three places and shadow each other.** Shell environment
beats `environment:` in the file, which beats `env_file:`, which beats the `.env`
file next to the compose file. `.env` is also the only one that substitutes
`${VAR}` inside the compose file itself. Details: `references/environments.md`.

## Workflow

1. **Inventory.** Which services, which are built here and which are images,
   what talks to what, what needs persistence.
2. **Write it,** starting from the block for each dependency in
   `references/datastores.md` / `references/brokers.md` and the application
   pattern in `references/apps.md`.
3. **`docker compose config`.** It resolves variables, merges overrides and
   validates the schema — a two-second check that catches most typos, including
   the ones the file *appears* to survive.
4. **`docker compose up -d`, then `docker compose ps`.** Look at the state
   column: `healthy`, `starting`, `unhealthy`, `exited (1)`.
5. **`docker compose logs -f <service>`** for whichever isn't green.
6. **Verify connectivity from inside**, not from the host:
   `docker compose exec api sh -c 'nc -z db 5432 && echo ok'`.

## Rules

- Pin images to `major.minor` (`postgres:17-alpine`). `image: postgres` means a
  major upgrade arrives unannounced and refuses to open the old data directory.
- `restart: unless-stopped` for long-running services; `restart: "no"` for
  one-shot jobs. `always` will restart a container you deliberately stopped.
- Give every stateful service a named volume from the start. Adding one later
  means the existing data was in the container layer and is gone.
- Put a healthcheck on anything another service depends on. Nothing else needs one.
- Don't publish ports you don't reach from the host. Use `expose:` to document
  the port instead.
- Networks: Compose creates one and joins everything to it. Add explicit networks
  when you want isolation (a backend network with `internal: true` has no route
  out) or to join an external proxy network — that case is `devops:traefik`.
- One concern per service. A container running both nginx and the app can't be
  scaled, restarted or debugged separately.

## Then

- Postgres, MongoDB, Redis → `references/datastores.md`
- RabbitMQ, Kafka, NATS → `references/brokers.md`
- Go, FastAPI, React+nginx, and the awesome-compose examples taken apart →
  `references/apps.md`
- dev vs prod, overrides, profiles, env and secrets → `references/environments.md`
- Hostname routing and TLS in front of the stack → `devops:traefik`
- It's up but broken → `devops:container-debugging`
