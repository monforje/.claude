# Hardening: user, capabilities, filesystem

Ordered by how much each actually buys you. The first item is most of the value.

## Non-root

A container process running as root is root on the host if anything escapes, and
in the common case it just quietly writes root-owned files into your bind-mounted
source tree.

```dockerfile
# Debian/slim
RUN groupadd -r -g 10001 app && useradd --no-log-init -r -u 10001 -g app app
# Alpine
RUN addgroup -g 10001 -S app && adduser -u 10001 -S app -G app
USER 10001:10001
```

- **Explicit numeric UID/GID.** Kubernetes' `runAsNonRoot` checks the *numeric*
  user, and a named `USER app` can't be verified without inspecting the image.
- `--no-log-init` avoids a pathological sparse `/var/log/faillog` with high UIDs.
- `USER` goes **after** the `COPY`s that need to write, or use
  `COPY --chown=app:app`. Files copied before `USER` are owned by root and a
  non-root process can't modify them.
- Ports below 1024 need privileges. Listen on 8080, publish as `-p 80:8080`.
- Bind mounts in dev: the container UID must match the host user or the files it
  creates are root-owned. Pass it in — see the Taskfile skill's
  `docker-patterns.md` (`DOCKER_UID`/`DOCKER_GID`).
- Distroless images ship a `nonroot` user (UID 65532): `USER nonroot:nonroot`.

## Runtime restrictions (Compose / `docker run`)

These are set where the container runs, not in the Dockerfile:

```yaml
services:
  api:
    read_only: true
    tmpfs: [/tmp]
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    user: "10001:10001"
```

- `no-new-privileges` blocks setuid escalation and costs nothing. Set it broadly.
- `cap_drop: [ALL]` then add back only what's needed (`cap_add: [NET_BIND_SERVICE]`
  to bind :80). Most application containers need none.
- `read_only: true` needs a writable `tmpfs` for anything that writes temp files;
  find out with `docker run --read-only` and read the errors.
- Never `privileged: true`. It disables essentially all of the above at once.

## Base image surface

- Fewest packages wins: `distroless` (no shell, no package manager) > `-slim` >
  `-alpine` > full distro. No shell also means no `docker exec sh` and no
  shell-based `HEALTHCHECK` — a real trade-off, decide per service.
- Pin by digest where a reproducible rebuild matters:
  `FROM alpine:3.21@sha256:...`. Automate bumps with Dependabot
  (`package-ecosystem: "docker"`) so pinning doesn't mean never updating.
- Scan before shipping: `docker scout quickview <image>` and
  `docker scout cves <image>`, or `trivy image <image>`. Both are advisory —
  a CVE in a package your code never calls is not an emergency.

## Secrets at runtime

Environment variables are visible to anything that can run `docker inspect` and
end up in logs and crash dumps. Prefer file-based secrets, which the official
images support directly:

```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db-password
    secrets: [db-password]
secrets:
  db-password:
    file: ./db/password.txt
```

Keep that file out of git. `awesome-compose/nginx-golang-postgres` commits a
`db/password.txt` — fine for a demo, wrong in a real repository.

## What not to bother with

- `sudo` in an image — unpredictable signal and TTY behaviour. Use `USER`, or
  `gosu` in an entrypoint if you must drop privileges at runtime.
- Squashing layers to "hide" a secret. The secret is in the build history.
- Chasing a zero-CVE base image. Fix the ones reachable from your code.
