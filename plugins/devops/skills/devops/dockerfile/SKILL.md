---
name: dockerfile
description: Write and optimise Dockerfiles — multi-stage builds, layer ordering so a source edit doesn't reinstall dependencies, base image choice per language, .dockerignore, non-root users, healthchecks, BuildKit cache and secret mounts, and multi-arch builds with buildx. Use whenever an image is being written, a build is slow, an image is too large, a container runs as root, a secret would otherwise be baked into a layer, or a Dockerfile needs review. Covers Go, Python/FastAPI and Node/React specifically. Not for wiring several services together (use the compose skill) and not for diagnosing a build that errors out (use container-debugging).
---

# Dockerfile

An image is a cache key and a filesystem. Almost every Dockerfile problem is one
of two things: the layers are ordered so the cache never hits, or things that
were only needed to *build* the app are still in the image that *runs* it.

## The two rules everything else follows from

**Copy dependency manifests before source.** Docker reuses a layer only if the
instruction and its inputs are unchanged. `COPY . .` before installing means
every one-character source edit reinstalls every dependency.

```dockerfile
COPY go.mod go.sum ./          # changes rarely
RUN go mod download            # cached across almost every build
COPY . .                       # changes constantly
RUN go build -o /out/app .
```

**The last stage is the image.** Everything above it — compilers, headers, dev
dependencies, the source tree — is discarded unless explicitly copied forward.
That is what makes a 900 MB build produce a 12 MB image.

```dockerfile
FROM golang:1.25 AS build
...
FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Per-language stage-by-stage recipes: `references/languages.md`.

## Workflow

1. **Read the project first.** Language, package manager, lockfile, build
   command, listening port, and whether an image already exists. Match the
   existing conventions before improving them.
2. **Write `.dockerignore` before the Dockerfile.** It decides how large the
   build context is and therefore how often the cache is invalidated. At
   minimum: `.git`, `node_modules`, `.venv`, `__pycache__`, `dist`, `build`,
   `*.log`, `.env`. Without it, a local `node_modules` is uploaded to the daemon
   on every single build.
3. **Pick the base image** from `references/languages.md`. Pin to `major.minor`
   (`python:3.13-slim`, not `python` or `latest`). Digest pinning is for
   production and CI, where a byte-identical rebuild matters.
4. **Write the build stage,** dependencies first, then source.
5. **Write the runtime stage:** copy only artefacts, set `WORKDIR` (absolute
   path), `USER`, `EXPOSE`, `ENTRYPOINT`/`CMD` in exec form.
6. **Build it.** `docker build -t x .` twice — the second run should be almost
   entirely cached. If it isn't, the ordering is wrong.
7. **Check the result:** `docker images x` for size, `docker run --rm x id` for
   the user, `docker history x` for which layer is fat.

## Rules

- **Exec form, always.** `CMD ["node", "server.js"]`, not `CMD node server.js`.
  Shell form wraps the process in `/bin/sh -c`, which does not forward SIGTERM,
  so the container ignores `docker stop` and gets killed after 10 seconds.
- **`ENTRYPOINT` for the program, `CMD` for its default arguments.** That keeps
  `docker run img --flag` working.
- **One `RUN` for `apt-get update && apt-get install`.** Split across two `RUN`s,
  the update layer gets cached and you install month-old package lists. End with
  `rm -rf /var/lib/apt/lists/*` in the same `RUN`.
- **`COPY`, not `ADD`.** `ADD` auto-extracts tars and fetches URLs — surprising.
  Use `ADD` deliberately for a remote artefact with `--checksum=`.
- **Never `ENV SECRET=...` or `ARG` a token.** Build args and env vars are
  visible in `docker history` forever. Use `--mount=type=secret`
  (`references/cache-and-secrets.md`).
- **Run as non-root** with an explicit UID/GID. `references/hardening.md`.
- **`WORKDIR /abs/path`**, never `RUN cd x && ...`.
- **Pipes need `set -o pipefail`.** `RUN wget -O- url | tar xz` succeeds even
  when `wget` fails, because only the last exit code counts.

## Healthchecks

A `HEALTHCHECK` is what makes `depends_on: condition: service_healthy` work in
Compose, so it is worth the three lines.

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:8000/health || exit 1
```

`--start-period` is the field people omit: during it, failures don't count
against `--retries`, which is the difference between a slow-booting app being
"starting" and being restarted forever. Note that the check runs *inside* the
container — a distroless or scratch image has no shell, no `curl`, no `wget`, so
either ship a tiny static health binary or declare the healthcheck in Compose
instead.

## Then

- Slow builds, large caches, secrets, multi-arch → `references/cache-and-secrets.md`
- Non-root, capabilities, read-only filesystems → `references/hardening.md`
- Go / Python / Node recipes → `references/languages.md`
- The build errors out → the `devops:container-debugging` skill, its
  `build-failures.md` reference

Source: [Docker build best practices](https://docs.docker.com/build/building/best-practices/).
