# Build failures

## Read the failing step

BuildKit prints the step number, the command and the exit code:

```
 => ERROR [build 5/9] RUN npm ci                                          2.4s
------
 > [build 5/9] RUN npm ci:
0.9 npm error code EUSAGE
------
```

`[build 5/9]` names the stage and the instruction. Look at *that* line of the
Dockerfile, not the whole file.

To inspect the state at the point of failure, stop the build one step earlier
and open a shell in that stage:

```bash
docker build --target build -t dbg .     # stop at a named stage
docker run --rm -it dbg sh
```

Or make BuildKit drop you into a shell on failure — the fastest option when the
error is about a missing file:

```bash
BUILDKIT_PROGRESS=plain docker build . 2>&1 | tail -40   # full output, no collapsing
docker build --progress=plain --no-cache .               # same, and no cached steps
```

## "file not found" in COPY

Three causes, in order of frequency:

1. **The path is relative to the build context, not the Dockerfile.** With
   `docker build -f docker/Dockerfile .`, `COPY app ./` means `./app` from the
   context root, not from `docker/`.
2. **`.dockerignore` excludes it.** Check: an ignored file simply is not there.
   `docker build --no-cache --progress=plain .` and look at the transfer size —
   a suspiciously small context is the clue.
3. **The compose `context:` is not the directory you assume.**
   `docker compose config` prints the resolved context path.

```bash
docker build --no-cache -q -t ctx-probe - <<'DF'
FROM alpine
COPY . /ctx
RUN find /ctx -maxdepth 2 | head -50
DF
```

That prints exactly what the daemon received.

## "no such file or directory" when the file is obviously there

The classic false alarm on a `scratch`/`distroless` image: the message refers to
the **dynamic linker**, not your binary. A cgo-enabled Go binary needs libc.
Rebuild with `CGO_ENABLED=0`, or use `gcr.io/distroless/base`.

Same message with a shell script: CRLF line endings. `#!/bin/sh\r` is not a
valid interpreter path. Fix with `dos2unix` or a `.gitattributes` entry.

## Changes are not picked up

- `docker compose up` **reuses the existing image**. Use `up -d --build`, or
  `docker compose build --no-cache <svc>` when a layer is stale.
- The file is in `.dockerignore`.
- You edited a file that is copied *before* the layer you expected to change, and
  the layer you're watching is cached. `docker build --progress=plain` shows
  `CACHED` on every reused step — read which ones.
- In a multi-stage build, the runtime stage copies from the build stage. Editing
  a file that only the build stage reads changes nothing in the final image
  unless the artefact itself changed.

## The cache never hits

Every build reinstalls dependencies:

- `COPY . .` happens **before** the install step. Copy the manifest and lockfile
  first, install, then copy the source.
- No `.dockerignore`, so `.git` or `node_modules` is in the context and its
  churn invalidates the `COPY`.
- A timestamp or build argument changes on every build (`ARG BUILD_DATE`) early
  in the file. Move it as late as possible.
- Multi-arch or a different builder: `docker buildx ls`, and remember cache is
  per builder.

## Slow builds

```bash
docker build --progress=plain . 2>&1 | grep -E '^#[0-9]+ DONE' | sort -k3 -n
```

Then attack the slowest step:

- Package installs → cache mounts (`../../dockerfile/references/cache-and-secrets.md`).
- Large context → `.dockerignore`. Check with the `ctx-probe` snippet above.
- arm64 building on amd64 (or vice versa) under QEMU → cross-compile instead:
  `--platform=$BUILDPLATFORM` plus `TARGETOS`/`TARGETARCH`.
- Independent stages build in parallel automatically; a stage that depends on
  another serialises. Splitting a dependency install into its own stage often
  helps.

## Out of space during a build

```bash
docker system df                     # where it went: images / containers / volumes / cache
docker builder prune --filter until=168h    # build cache older than a week
docker image prune                   # dangling images only
```

`docker builder prune -a` and `docker system prune -a` remove far more, including
other projects' work. Use the filtered forms.

## Network failures inside a build

`apt-get`/`npm`/`pip` timing out during build but working on the host is usually
DNS in the build container. Try `docker build --network=host .` to confirm, then
fix the daemon's DNS (`/etc/docker/daemon.json` → `"dns": ["1.1.1.1"]`) rather
than leaving `--network=host` in place. Behind a corporate proxy, pass
`--build-arg HTTP_PROXY=... --build-arg HTTPS_PROXY=...`.
