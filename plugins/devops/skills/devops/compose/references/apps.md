# Application services, and the awesome-compose examples taken apart

The five [awesome-compose](https://github.com/docker/awesome-compose) examples,
what to copy from each and what to fix. They are demos, not production files —
several share the same defects (unpinned images, no healthchecks, `depends_on`
without a condition, and `dev-envs` stages that only exist for Docker Desktop's
dev environments feature and are pure noise in a normal project).

## `nginx-golang-postgres` — the best of the five

Take almost all of it. It is the only one that does startup ordering properly:

```yaml
  backend:
    build: { context: backend, target: builder }
    secrets: [db-password]
    depends_on:
      db: { condition: service_healthy }
  db:
    image: postgres
    user: postgres
    environment:
      - POSTGRES_DB=example
      - POSTGRES_PASSWORD_FILE=/run/secrets/db-password
    expose: ["5432"]
    volumes: [db-data:/var/lib/postgresql/data]
    healthcheck: { test: ["CMD", "pg_isready"], interval: 10s, timeout: 5s, retries: 5 }
secrets:
  db-password: { file: db/password.txt }
```

- **Copy:** `condition: service_healthy`, the password as a file secret rather
  than an env var, `expose:` instead of publishing Postgres to the host, the
  named volume, and the Dockerfile's `FROM scratch` final stage.
- **Fix:** `image: postgres` is unpinned — use `postgres:17-alpine`.
  `pg_isready` without `-U`/`-d` returns 0 too early (see `datastores.md`).
  `target: builder` ships the *build* stage — the whole point of that Dockerfile
  is the `scratch` stage below it, so drop the `target`. And `db/password.txt`
  is committed to the repo; keep yours out of git.

## `traefik-golang` — the label pattern, one line at a time

```yaml
  backend:
    build: backend
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.go.rule=Path(`/`)"
      - "traefik.http.services.go.loadbalancer.server.port=80"
```

- **Copy:** the three-part shape — enable, a router with a rule, a service with
  the container port. `--providers.docker.exposedbydefault=false` on the proxy.
- **Fix:** `traefik:2.6` is two majors behind (v3 changed rule syntax);
  `/var/run/docker.sock` is mounted writable — add `:ro`. Full treatment in
  `devops:traefik`.

## `react-nginx` — the SPA config worth memorising

The Dockerfile builds with Node and serves with nginx, which is right. The part
that is not obvious and *must* be copied verbatim:

```nginx
server {
  listen 80;
  location / {
    root   /usr/share/nginx/html;
    index  index.html;
    try_files $uri /index.html =404;
  }
}
```

Without `try_files`, `/` works and a reload on `/settings` returns 404 — nginx
looks for a file that a client-side router was supposed to handle. Every SPA
behind nginx needs this line.

- **Fix:** `FROM node:lts` is unpinned and `FROM development AS build` inherits
  the dev stage, so the build carries `CI=true`, `PORT` and dev deps. Use the
  two-stage version in `../../dockerfile/references/languages.md`. Delete the
  `dev-envs` stage.
- Its `.dockerignore` is good and short: `node_modules`, `build`, `**/.git`.

## `fastapi` — take the cache mount, leave the rest

- **Copy:** `RUN --mount=type=cache,target=/root/.cache/pip pip install -r
  requirements.txt`.
- **Fix:** the base image `tiangolo/uvicorn-gunicorn-fastapi` is deprecated by
  its own author; `requirements.txt` pins nothing; compose builds
  `target: builder` and publishes 8000 with no healthcheck. Replace with the
  `python:3.13-slim` recipe in `languages.md`.

## `nginx-nodejs-redis` — a load-balancer demo, not a template

Two identical `web1`/`web2` services differing only by `hostname:`, an nginx
`upstream` in front, Redis published on 6379 with no volume and no healthcheck.
The idea (nginx round-robins over two app containers) is fine; as a stack file
it is missing everything in the SKILL's "four things" list. Use it to read the
`upstream` block, not to copy the compose file.

## Application service, generally

```yaml
  api:
    build:
      context: .
      target: runtime
      args: { APP_VERSION: "${APP_VERSION:-dev}" }
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db:    { condition: service_healthy }
      redis: { condition: service_healthy }
    ports: ["8000:8000"]
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:8000/health || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 20s
```

- **Bind to `0.0.0.0`.** Uvicorn, Flask's dev server, `next dev` and many others
  default to `127.0.0.1`, which inside a container is unreachable from anywhere.
- Point the frontend at the backend through the proxy or via the browser's
  origin — a React bundle runs in the *browser*, so `http://api:8000` in
  frontend code resolves to nothing. Only server-side code can use service names.
- Development hot reload:

```yaml
    develop:
      watch:
        - { action: sync, path: ./src, target: /app/src }
        - { action: rebuild, path: package.json }
```

  `docker compose watch` syncs changed files without a rebuild and rebuilds only
  when the manifest changes. Cleaner than bind-mounting the whole tree, and it
  avoids the classic "host `node_modules` shadows the container's" breakage.
