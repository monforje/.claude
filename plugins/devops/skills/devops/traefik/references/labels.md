# Label grammar

Every label is `traefik.<protocol>.<kind>.<your-name>.<setting>=<value>`.
`<your-name>` is an identifier you invent; the same name in a router label and a
service label is what binds them together.

```
traefik.http.routers.myapi.rule=Host(`api.localhost`)
traefik.http.routers.myapi.entrypoints=websecure
traefik.http.routers.myapi.tls=true
traefik.http.routers.myapi.middlewares=strip-api@docker
traefik.http.services.myapi.loadbalancer.server.port=8000
traefik.http.middlewares.strip-api.stripPrefix.prefixes=/api
```

- `routers` — matches requests. `services` — where they go. `middlewares` —
  what happens in between.
- Names are global across the whole provider, not per container. Two projects
  both using `traefik.http.routers.api.*` collide; prefix with the project name.
- Referencing a middleware defined in the same provider needs the provider
  suffix: `@docker` (or `@file` for one from a file provider). Within a single
  container's labels you can omit it, but writing it is never wrong.

## List form vs map form

Compose accepts either. Both are valid; pick one per file.

```yaml
    labels:
      - "traefik.http.routers.api.rule=Host(`api.localhost`)"    # list
    labels:
      traefik.http.routers.api.rule: "Host(`api.localhost`)"     # map
```

Two things bite here:

- **Backticks in rules are required by Traefik** and must survive YAML. Quote the
  whole value. Single quotes inside a rule are not accepted by Traefik v3.
- **`$` must be doubled in Compose.** Compose interpolates `${...}` and `$x`
  before Docker ever sees the label, and bcrypt hashes are full of `$`:

```yaml
      - "traefik.http.middlewares.auth.basicauth.users=test:$$2y$$12$$ci4U63YX..."
```

  A single `$` gives `variable is not set` warnings and a hash that never matches.
  This does not apply when Traefik reads a file provider — only to Compose labels.

## Option names are case-insensitive, values are not

`stripPrefix`, `stripprefix` and `StripPrefix` all work; Traefik normalises them.
Router and middleware *names* and rule values are case-sensitive. Docs use
camelCase — follow that for readability, not correctness.

## Rules (v3 syntax)

```
Host(`api.localhost`)
Host(`a.localhost`) || Host(`b.localhost`)
Host(`api.localhost`) && PathPrefix(`/v1`)
HostRegexp(`^(.+\.)?example\.local$`)      # v3: Go regex, anchored yourself
PathPrefix(`/api`) && !Path(`/api/health`)
Method(`GET`) || Method(`HEAD`)
Headers(`X-Env`, `staging`)
ClientIP(`10.0.0.0/8`)
```

v3 changed this from v2: `Host` no longer takes multiple comma-separated values,
and `HostRegexp` uses Go regular expressions instead of the old `{name:pattern}`
syntax. Copying a v2 example is the usual reason a rule silently matches nothing.

**Priority.** Longest rule wins by default. When two routers overlap, set it
explicitly: `traefik.http.routers.api.priority=100`.

## Services

```
traefik.http.services.api.loadbalancer.server.port=8000
traefik.http.services.api.loadbalancer.server.scheme=https   # backend speaks TLS
traefik.http.services.api.loadbalancer.passhostheader=true   # default
traefik.http.services.api.loadbalancer.sticky.cookie.name=srv
traefik.http.services.api.loadbalancer.healthcheck.path=/health
traefik.http.services.api.loadbalancer.healthcheck.interval=10s
```

The port is the **container's** port, never the published one. Scaling
(`docker compose up -d --scale api=3`) load-balances across replicas
automatically — nothing extra to declare.

## Two routers on one container

The common case is http and https for the same host, where the http one only
redirects:

```yaml
      - "traefik.http.routers.api-http.rule=Host(`api.localhost`)"
      - "traefik.http.routers.api-http.entrypoints=web"
      - "traefik.http.routers.api-http.middlewares=api-https"
      - "traefik.http.middlewares.api-https.redirectscheme.scheme=https"
      - "traefik.http.routers.api.rule=Host(`api.localhost`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls=true"
```

Same rule, different entrypoints, different names. See `middlewares.md` for when
to do this per-domain versus globally on the entrypoint.

## TCP and UDP

Non-HTTP services (Postgres, FTP, game servers) route by SNI, not by host header.
`HostSNI(`*`)` is the catch-all and is the only rule allowed without TLS.

```yaml
      - "traefik.tcp.routers.pg.rule=HostSNI(`*`)"
      - "traefik.tcp.routers.pg.entrypoints=postgres"
      - "traefik.tcp.services.pg.loadbalancer.server.port=5432"
```

with `--entrypoints.postgres.address=:5432` on Traefik. One TCP entrypoint per
port: `HostSNI(`*`)` matches everything, so two such routers on one entrypoint
conflict.

## Traefik's own dashboard

The API is an internal service; reference it rather than a port:

```yaml
      - "traefik.http.routers.dashboard.rule=Host(`traefik.localhost`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=dashboard-auth"
```
