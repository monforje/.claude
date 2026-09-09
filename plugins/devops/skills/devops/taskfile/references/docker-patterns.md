# Task + Docker: the recurring shapes

## Idempotent resources

The pattern behind everything else: a `status:` check that costs nothing, a
command that creates the thing.

```yaml
  network:
    desc: "Create the shared proxy network"
    internal: true
    status: ["docker network inspect traefik"]
    cmds: ["docker network create traefik"]

  volume:
    internal: true
    status: ["docker volume inspect myapp-certs"]
    cmds: ["docker volume create myapp-certs"]

  certs:
    desc: "Issue the local TLS certificate"
    internal: true
    deps: [volume]
    status:
      - "docker run --rm -v myapp-certs:/certs alpine:3.21 test -f /certs/dynamic.yml"
    cmds:
      - >
        docker run --rm -v myapp-certs:/certs -e CAROOT=/certs -e HOME=/certs
        golang:1.25-alpine go run filippo.io/mkcert@latest
        -cert-file /certs/myapp.local.pem -key-file /certs/myapp.local-key.pem
        myapp.local "*.myapp.local"
```

Each `status:` command must have **no side effects and need no privileges** — it
runs on every invocation. A check that needs `sudo` turns a no-op `task up` into
a password prompt.

## Host UID/GID into the container

A container writing into a bind mount creates files owned by whatever UID it runs
as. Root by default — so the dev server leaves root-owned `node_modules`,
`.venv` or build output in your source tree, and the next non-Docker command
fails with permission errors.

```yaml
env:
  DOCKER_UID: { sh: id -u }
  DOCKER_GID: { sh: id -g }
```

```yaml
# compose.yaml
services:
  web:
    user: "${DOCKER_UID:-1000}:${DOCKER_GID:-1000}"
    volumes: ["./:/app"]
```

Declared as `env:` (not `vars:`) so Compose can interpolate it.

## A tool you don't install on the host

Run it from a throwaway container instead. Useful for anything needed once —
mkcert, `certutil`, migration tools, `psql`.

```yaml
  migrate:
    desc: "Apply database migrations"
    cmds:
      - >
        docker run --rm --user "{{.DOCKER_UID}}:{{.DOCKER_GID}}"
        -v ./migrations:/migrations --network myapp_default
        arigaio/atlas:latest migrate apply --dir file:///migrations
        --url "postgres://db:5432/app?sslmode=disable"
```

`--network` must be the network Compose actually created:
`<project>_default`, where the project is `name:` in the compose file or the
directory name. `docker network ls` confirms it.

## Waiting for health

`docker compose up -d` returns as soon as containers are created. If the next
task needs a working database, wait for it:

```yaml
  wait-db:
    internal: true
    cmds:
      - >
        timeout 60 sh -c 'until [ "$(docker inspect -f {{`{{.State.Health.Status}}`}} myapp-db)" = healthy ];
        do sleep 1; done' || { echo "db never became healthy"; docker compose logs db; exit 1; }
```

Note the escaped `{{` — Go templating would otherwise eat the Docker format
string. Better still, let Compose wait via `depends_on: condition:
service_healthy`; this task is for steps running *outside* the stack.

## Logs and exec with arguments

```yaml
  logs:
    desc: "Follow logs: task logs -- api"
    cmds: ["docker compose logs -f --tail 100 {{.CLI_ARGS}}"]

  sh:
    desc: "Shell into a service: task sh -- api"
    interactive: true
    cmds: ["docker compose exec {{.CLI_ARGS}} sh"]
```

`interactive: true` is required for anything with a TTY, or the shell hangs with
no prompt.

## Destructive tasks, named and guarded

```yaml
  down:
    desc: "Stop the stack (data survives)"
    cmds: ["docker compose down"]

  nuke:
    desc: "Stop the stack AND DELETE ALL DATA"
    prompt: "This deletes every volume in this project. Continue?"
    cmds: ["docker compose down -v"]
```

Never let `down -v` be reachable from `up`, and never put it in a `deps:` chain.

## A `deploy` that is safe to rerun

The whole point of `status:` guards is that this converges from any state:

```yaml
  deploy:
    desc: "Bring everything up from scratch"
    cmds:
      - task: init          # copy .env.example where .env is missing
      - task: network       # skipped if it exists
      - task: certs         # skipped if issued
      - task: up
      - task: migrate
      - task: where         # print the URLs
```

Sequential `cmds:`, not `deps:` — order matters here and `deps:` would run them
all at once.
