---
name: container-debugging
description: Diagnose Docker failures — a build that errors out or silently ignores changed files, a container that exits immediately or restarts forever, exit codes 125/126/127/137/143, a healthcheck stuck unhealthy, two services that can't reach each other by name, port conflicts, permission-denied on a mounted volume, disk filling up, and slow builds. Use whenever something is already written and not working: "docker build fails", "the container won't start", "it exits with 137", "connection refused between services", "permission denied", "my changes aren't picked up". Not for writing a new Dockerfile or compose file from scratch.
---

# Container debugging

Diagnosis first. Almost every Docker failure announces itself precisely, in a
place people skip past.

## Read the actual error

```bash
docker compose ps                     # state: healthy / starting / unhealthy / exited (N)
docker compose logs --tail 100 <svc>  # the app's own output
docker inspect <container> --format '{{.State.ExitCode}} {{.State.Error}}'
docker inspect <container> --format '{{json .State.Health}}' | jq
docker events --since 10m             # what the daemon did, in order
```

An exited container still exists: `docker compose logs` reads a dead container's
output just fine, and `docker compose ps -a` lists it. "The container vanished"
almost always means it exited and was removed by `--rm`.

## Split the problem in one step

**Does it build?** `docker build .` → build problem, `references/build-failures.md`.

**Does it start?** `docker run --rm -it <image> sh` → if the shell works, the
image is fine and the problem is the command, the environment, or the mounts.

**Does it start under Compose but not do its job?** → runtime problem,
`references/runtime-failures.md`.

**Bypass your entrypoint to look inside:**

```bash
docker run --rm -it --entrypoint sh <image>
docker compose run --rm --entrypoint sh <service>
```

That last one keeps the compose environment, networks and mounts while giving
you a shell — the single most useful debugging command in Compose.

## Exit codes

| Code | Meaning | Look at |
| --- | --- | --- |
| 0 | Finished normally | The command is not a long-running process |
| 1 | Application error | `docker compose logs` — the app's own message |
| 125 | The daemon rejected `docker run` | Your flags, not the image |
| 126 | Command found but not executable | Missing `chmod +x`, or CRLF line endings |
| 127 | Command not found | Wrong path, or a binary missing from a slim image |
| 137 | SIGKILL — usually OOM | `docker inspect --format '{{.State.OOMKilled}}'`; raise the memory limit |
| 139 | Segfault | Often an architecture mismatch (arm64 image on amd64) |
| 143 | SIGTERM — a normal stop | Nothing wrong |

**137 is the one to recognise.** A container killed for memory looks exactly
like a crash, and the application log usually ends mid-sentence with no error.

## Rules while debugging

- **Change one thing, then re-run.** Docker failures cascade; three simultaneous
  edits make the actual cause unattributable.
- **`docker compose config` before believing the file.** It shows what Compose
  really resolved after variables and overrides. Frequently the answer is there.
- **Never `docker system prune` to "clean up"** — it deletes other projects'
  images, networks and build cache. If disk space is genuinely the problem, see
  `references/runtime-failures.md` for the targeted commands.
- **Never `down -v`** to reset a stuck stack. That deletes the data. `down` then
  `up -d` recreates every container without touching volumes.
- **Say when the cause is outside Docker.** A container that starts and then
  returns 500s is an application bug; the Docker question is answered.

## The five things it usually is

1. **The app listens on `127.0.0.1`.** Inside a container that means nothing can
   reach it. Must be `0.0.0.0`. Symptom: works locally, "connection refused"
   through Docker.
2. **`depends_on` without `condition: service_healthy`.** The app starts before
   the database accepts connections and exits. Symptom: works on the second
   `up`, fails on the first.
3. **Wrong hostname or port between services.** Use the *service name* and the
   *container* port (`db:5432`), never `localhost` and never the published port.
4. **The build cache served a stale layer.** Changed a file, nothing happened:
   either it is excluded by `.dockerignore`, or you are running an old image
   because `docker compose up` without `--build` reuses it.
5. **Volume permissions.** The container's UID cannot write to a bind-mounted
   host directory, or a named volume was initialised by a root process earlier.

## Then

- Build errors, cache misses, context problems, slow builds →
  `references/build-failures.md`
- Crashes, restart loops, healthchecks, networking, volumes, ports, disk →
  `references/runtime-failures.md`
- The fix means rewriting the image → `devops:dockerfile`
- The fix means rewriting the stack → `devops:compose`
- 404 or 502 from a proxy → `devops:traefik`
