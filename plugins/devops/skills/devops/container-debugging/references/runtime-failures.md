# Runtime failures

## Exits immediately

```bash
docker compose ps -a                             # the dead container is still listed
docker compose logs --tail 50 <svc>
docker inspect <c> --format '{{.State.ExitCode}} {{.State.Error}} {{.State.OOMKilled}}'
```

- **Exit 0, no error** — the command finished. A container lives exactly as long
  as PID 1. `command: npm run build` exits when the build ends; that is correct
  behaviour, not a crash. A one-shot service needs `restart: "no"`.
- **Exit 1** — read the application's own log. This is not a Docker problem.
- **Exit 127** — command not found. Common after switching to a slim or
  distroless base that has no shell: `CMD ["sh", "-c", ...]` cannot work there.
- **Exit 126** — found but not executable. `chmod +x` before `COPY`, or CRLF
  line endings in the entrypoint script.
- **Exit 137 with `OOMKilled: true`** — the memory limit. Raise
  `deploy.resources.limits.memory`, or make the process use less. JVM and Node
  read the *host's* memory unless told otherwise (`-XX:MaxRAMPercentage`,
  `--max-old-space-size`).
- **Exit 137 with `OOMKilled: false`** — something sent SIGKILL: usually
  `docker stop` timing out after 10s because PID 1 ignores SIGTERM. That happens
  with shell-form `CMD`; use exec form.

## Restart loop

`restart: unless-stopped` plus a crash on startup gives an endless loop that
also hides the error, because each new container replaces the last.

```bash
docker compose logs --tail 200 <svc>     # logs from previous runs are retained
docker compose stop <svc>                # stop the loop, then investigate
docker compose run --rm --entrypoint sh <svc>   # same env and mounts, manual start
```

## Healthcheck stuck unhealthy

```bash
docker inspect <c> --format '{{json .State.Health}}' | jq '.Log[-1]'
```

That prints the last probe's exit code and output — normally the answer outright.

- The check runs **inside** the container. `curl` is not in `python:3-slim`,
  `nginx:alpine` has `wget` but not `curl`, distroless has neither. Test with
  `docker compose exec <svc> <the exact command>`.
- Use `127.0.0.1`, not the service name — a container reaching itself by service
  name goes out to the Docker DNS and back.
- No `start_period`, so a slow boot burns all the retries and the container is
  restarted forever. Set `start_period: 30s`.
- `test:` as a bare string is run with `/bin/sh -c`; as a list it must begin with
  `CMD` or `CMD-SHELL`. A list starting with the binary name silently never runs.

## Services can't reach each other

```bash
docker compose exec api sh -c 'nc -z db 5432 && echo open || echo closed'
docker compose exec api getent hosts db
docker network inspect <project>_default --format '{{range .Containers}}{{.Name}} {{end}}'
```

- **Name doesn't resolve** — the two containers are on different networks, or you
  used the container name instead of the service name. Compose resolves service
  names on its own networks only.
- **Resolves but connection refused** — the app is bound to `127.0.0.1` inside
  its container. Bind `0.0.0.0`. This is the single most frequent cause.
- **Wrong port** — use the container port (`db:5432`), not the published one.
  Publishing is only about the host.
- **`localhost` from a container is that container.** To reach a service running
  on the host machine, use `host.docker.internal` (add
  `extra_hosts: ["host.docker.internal:host-gateway"]` on Linux).
- An `internal: true` network has no outbound route — deliberate for a database,
  fatal for a service that must call an external API.

## Port already in use

```
Error starting userland proxy: listen tcp4 0.0.0.0:80: bind: address already in use
```

```bash
sudo ss -lptn 'sport = :80'          # what holds it
docker ps --filter publish=80        # a container of yours?
```

Usually a proxy from another project, or a host nginx. Change your published
port; do **not** kill the other container without checking what it belongs to.

## Permission denied on a volume

- **Bind mount**: the container's UID must be able to write to the host
  directory. `docker compose exec <svc> id` shows who it runs as. Fix by running
  the container as your UID (`user: "${UID}:${GID}"`, see the Taskfile skill's
  `docker-patterns.md`), not by `chmod 777` on your source tree.
- **Named volume**: on first use Docker copies the image's directory contents
  *and ownership* into the empty volume. If that happened while the image still
  ran as root, a later non-root process can't write. Recreate that one volume:
  `docker volume rm <project>_<name>` (it is empty of your data if the service
  never worked).
- **SELinux** (Fedora/RHEL): add `:z` to the mount — `./src:/app:z`.
- Root-owned files appearing in your source tree is the same problem from the
  other side: the container ran as root and wrote there.

## Nothing in the logs

- The application buffers stdout. `PYTHONUNBUFFERED=1`, `stdbuf -o0`, or the
  language's equivalent.
- It logs to a file inside the container instead of stdout. Point it at stdout;
  that is the container convention.
- You are reading the wrong container: `docker compose logs` with no argument
  shows everything, prefixed.

## Disk filling up

```bash
docker system df                      # and `-v` for the per-object breakdown
docker builder prune --filter until=168h
docker image prune                    # dangling only
docker volume ls -qf dangling=true    # LOOK before removing — this may be data
```

Also check log size: a container with no rotation can produce gigabytes.
Set it once, globally, in `/etc/docker/daemon.json`:

```json
{ "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "3" } }
```

Never reach for `docker system prune -a` on a shared machine.
