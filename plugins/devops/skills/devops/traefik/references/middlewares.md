# Middlewares and Let's Encrypt

A middleware is declared once and attached by name. Order matters: they run
left to right as listed on the router.

```yaml
      - "traefik.http.routers.api.middlewares=api-auth@docker,api-strip@docker"
```

## Redirects

**http → https, globally on the entrypoint** — simplest, right for a machine
hosting one domain:

```yaml
    command:
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
```

**Per domain** — necessary when other hostnames on the same machine must keep
working over plain http (internal webhooks do not follow redirects):

```yaml
      - "traefik.http.routers.api-http.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api-http.entrypoints=web"
      - "traefik.http.routers.api-http.middlewares=api-https"
      - "traefik.http.middlewares.api-https.redirectscheme.scheme=https"
      - "traefik.http.middlewares.api-https.redirectscheme.permanent=true"
```

`permanent=true` sends 308 instead of 302 — cached hard by browsers, so only set
it once the setup is settled.

**www → non-www**, and redirecting to an external URL:

```yaml
      - "traefik.http.middlewares.nowww.redirectregex.regex=^https?://www\\.(.+)"
      - "traefik.http.middlewares.nowww.redirectregex.replacement=https://$${1}"
      - "traefik.http.middlewares.nowww.redirectregex.permanent=true"
```

`$${1}` — one `$` for the regex group, doubled because Compose eats the other.

## Basic auth

```bash
htpasswd -nbB admin 'password' | sed -e 's/\$/\$\$/g'
```

```yaml
      - "traefik.http.middlewares.dash-auth.basicauth.users=admin:$$2y$$05$$..."
```

The `sed` doubles every `$` for Compose. Alternatively put the untouched hash in
a file and use `basicauth.usersfile=/etc/traefik/users` with a file provider —
no escaping, and the hash stays out of the compose file.

## Prefix handling

```yaml
      - "traefik.http.middlewares.strip.stripprefix.prefixes=/api"
      - "traefik.http.middlewares.addpre.addprefix.prefix=/v1"
      - "traefik.http.middlewares.rewrite.replacepathregex.regex=^/api/(.*)"
      - "traefik.http.middlewares.rewrite.replacepathregex.replacement=/$${1}"
```

`stripPrefix` is what you want when a backend is mounted at `/api` by the proxy
but serves from `/` itself. If the backend generates absolute URLs it will still
emit `/` paths that break — that is an application problem, not a Traefik one.

## Headers, CORS, rate limiting, retries

```yaml
      - "traefik.http.middlewares.sec.headers.stsseconds=31536000"
      - "traefik.http.middlewares.sec.headers.framedeny=true"
      - "traefik.http.middlewares.sec.headers.contenttypenosniff=true"
      - "traefik.http.middlewares.cors.headers.accesscontrolalloworiginlist=https://app.example.com"
      - "traefik.http.middlewares.cors.headers.accesscontrolallowmethods=GET,POST,OPTIONS"
      - "traefik.http.middlewares.cors.headers.accesscontrolmaxage=100"
      - "traefik.http.middlewares.rl.ratelimit.average=100"
      - "traefik.http.middlewares.rl.ratelimit.burst=50"
      - "traefik.http.middlewares.retry.retry.attempts=3"
      - "traefik.http.middlewares.ip.ipallowlist.sourcerange=10.0.0.0/8,192.168.0.0/16"
```

Only set HSTS (`stsseconds`) once https definitely works — browsers cache it and
will refuse plain http for that domain for a year.

Behind another proxy or a CDN, `ipallowlist` sees the wrong address until you
set `--entrypoints.web.forwardedheaders.trustedips=<the upstream's IP>`.

## Chains

```yaml
      - "traefik.http.middlewares.protected.chain.middlewares=sec@docker,dash-auth@docker,rl@docker"
      - "traefik.http.routers.dashboard.middlewares=protected@docker"
```

## Let's Encrypt

**HTTP challenge** — the default. Needs port 80 reachable from the internet and
a real DNS record pointing at the machine.

```yaml
    command:
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.le.acme.httpchallenge=true"
      - "--certificatesresolvers.le.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.le.acme.email=you@example.com"
      - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
    volumes:
      - ./letsencrypt:/letsencrypt
```

```yaml
      - "traefik.http.routers.api.tls=true"
      - "traefik.http.routers.api.tls.certresolver=le"
```

**DNS challenge** — required for wildcards, and the only option when port 80 is
not reachable. Traefik creates a TXT record through your DNS provider's API:

```yaml
      - "--certificatesresolvers.le.acme.dnschallenge=true"
      - "--certificatesresolvers.le.acme.dnschallenge.provider=cloudflare"
    environment:
      CF_DNS_API_TOKEN: ${CF_DNS_API_TOKEN}
```

```yaml
      - "traefik.http.routers.api.tls.certresolver=le"
      - "traefik.http.routers.api.tls.domains[0].main=example.com"
      - "traefik.http.routers.api.tls.domains[0].sans=*.example.com"
```

Practicalities:

- `acme.json` must be persisted and mode 600, or Traefik refuses to start. Losing
  it means re-issuing everything.
- **Use the staging server while setting up:**
  `--certificatesresolvers.le.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory`.
  Production is rate-limited to 5 duplicate certificates per week, and a
  misconfigured loop burns that in minutes. Delete `acme.json` when switching to
  production, or the staging certificates are kept.
- Wildcards are DNS challenge only. There is no HTTP-challenge wildcard.
- For a local domain none of this applies — use mkcert, `tls-local.md`.
