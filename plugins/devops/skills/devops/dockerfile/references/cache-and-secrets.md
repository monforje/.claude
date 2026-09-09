# BuildKit: cache mounts, secrets, multi-arch

All of this needs BuildKit, which is the default in current Docker. Add
`# syntax=docker/dockerfile:1` as the first line to get the newest frontend
features regardless of the daemon's age.

## Cache mounts

A cache mount is a directory that persists **between builds** but is not part of
any layer. It is the difference between "the layer cache was invalidated so
re-download 400 MB" and "re-download the three packages that changed".

```dockerfile
RUN --mount=type=cache,target=/root/.npm        npm ci
RUN --mount=type=cache,target=/root/.cache/pip  pip install -r requirements.txt
RUN --mount=type=cache,target=/root/.cache/uv   uv sync --frozen --no-dev
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build  go build ./...
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends curl
```

- The apt variant needs `sharing=locked` (parallel builds would corrupt the
  lists) and you must *not* `rm -rf /var/lib/apt/lists/*` — the mount is already
  outside the layer. Also drop the image's `docker-clean` config first:
  `RUN rm -f /etc/apt/apt.conf.d/docker-clean`.
- Cache mounts are local to the machine. They speed up your laptop and a
  persistent CI runner; on ephemeral CI use registry cache instead:
  `docker buildx build --cache-to type=registry,ref=user/app:cache,mode=max
  --cache-from type=registry,ref=user/app:cache .`
- Clearing them: `docker builder prune --filter type=exec.cachemount`.

## Bind mounts instead of COPY

For a file needed only during one `RUN`, mount it rather than adding a layer:

```dockerfile
RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    pip install -r /tmp/requirements.txt
```

## Secrets

**Never** `ARG NPM_TOKEN` / `ENV API_KEY=...`. Both are recorded in the image
metadata; `docker history --no-trunc <image>` prints them back out, and squashing
or deleting the file in a later layer does not remove it.

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
RUN --mount=type=secret,id=api_key \
    API_KEY="$(cat /run/secrets/api_key)" && ./fetch-private-deps.sh
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
docker build --secret id=api_key,env=API_KEY .
```

The default mount path is `/run/secrets/<id>`; `target=` overrides it. Nothing
lands in any layer. In Compose:

```yaml
services:
  api:
    build:
      context: .
      secrets: [npmrc]
secrets:
  npmrc:
    file: ~/.npmrc
```

SSH for private Git dependencies: `RUN --mount=type=ssh git clone git@...` built
with `docker build --ssh default .`.

## Multi-arch with buildx

```bash
docker buildx create --name multi --use --bootstrap
docker buildx build --platform linux/amd64,linux/arm64 -t user/app:1.2.3 --push .
```

- `--push` is effectively required: a multi-platform result is a manifest list,
  and the local image store can't hold one (`--load` only works for a single
  platform).
- Use the pair `FROM --platform=$BUILDPLATFORM golang:1.25 AS build` plus
  `ARG TARGETOS TARGETARCH` and `GOOS=$TARGETOS GOARCH=$TARGETARCH go build`.
  That cross-compiles on the native builder instead of emulating the target
  under QEMU, which for Go is roughly an order of magnitude faster.
- Interpreted stacks (Python, Node) can't cross-compile native extensions this
  way and will fall back to emulation — expect slow arm64 builds on x86.

## Rebuilding for security

`docker build` reuses the cached base image forever. `--pull` fetches a newer
base, `--no-cache` re-runs every instruction; together they produce a genuinely
fresh image. Worth doing on a schedule, not on every build.
