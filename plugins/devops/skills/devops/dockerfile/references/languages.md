# Base images and recipes per language

Pinned to `major.minor`. Bump deliberately; don't use `latest`.

## Go — build on `golang`, ship on `distroless/static`

Go links statically, so the runtime image needs nothing but the binary. This is
the one stack where a ~10 MB final image is normal.

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.25-alpine AS build
WORKDIR /src
ENV CGO_ENABLED=0
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -ldflags="-s -w" -o /out/app ./cmd/api

FROM gcr.io/distroless/static:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

- `CGO_ENABLED=0` is what makes the binary static. With cgo on, `distroless/static`
  and `scratch` fail at startup with "no such file or directory" — a message that
  is really "no dynamic linker". Use `gcr.io/distroless/base` if you need cgo.
- `-ldflags="-s -w"` drops the symbol table: a few MB, no runtime cost.
- `scratch` works too, but has no CA bundle — outbound HTTPS fails until you
  `COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/`.
  `distroless/static` already includes certs, timezone data and a nonroot user.

## Python / FastAPI — `python:3.x-slim`, not alpine

**Alpine is the wrong default for Python.** It uses musl instead of glibc, so
any dependency without a musl wheel (numpy, pandas, psycopg2, pydantic-core,
cryptography) is compiled from source: builds go from seconds to many minutes
and the image often ends up *larger* than the slim one.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim AS build
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install -r requirements.txt

FROM python:3.13-slim
WORKDIR /app
RUN useradd --no-log-init -r -u 10001 app
COPY --from=build /install /usr/local
COPY --chown=app:app ./app ./app
USER 10001
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- `--host 0.0.0.0`. Uvicorn defaults to `127.0.0.1`, which inside a container
  means "reachable by nothing". This is the number one FastAPI-in-Docker bug.
- `PYTHONUNBUFFERED=1` or your logs stay in a buffer and `docker logs` looks empty.
- With uv: `COPY pyproject.toml uv.lock ./` then
  `RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-dev`.
- **Don't use `tiangolo/uvicorn-gunicorn-fastapi`** (as `awesome-compose/fastapi`
  does). Its own author deprecated it; plain `python:3.x-slim` + uvicorn is the
  current recommendation.

## Node / React — build with node, serve with nginx

The final image contains no Node at all: a React build is static files.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

- `npm ci`, not `npm install` — it obeys the lockfile and fails if it drifted.
- Output dir is `dist` for Vite, `build` for Create React App.
- The nginx config is not optional for a SPA: see `../../compose/references/apps.md`
  for the `try_files` line that stops a page reload returning 404.
- Node server (not static): `node:22-alpine` for both stages, `npm ci --omit=dev`
  in the runtime stage, `USER node`, and exec-form `CMD ["node", "dist/index.js"]`.
  `--only=production` is deprecated; `--omit=dev` replaced it.
