---
name: traefik
description: Configure Traefik v3 as a Docker reverse proxy — routers, services and middlewares declared as container labels, hostname routing, entrypoints, local https with mkcert, Let's Encrypt HTTP and DNS challenges, redirects, basic auth, prefix stripping, and a shared external proxy network for several projects on one machine. Use whenever a container should answer on a hostname, https is needed locally or in production, a Traefik route returns 404 or 502 or hits the wrong service, or labels need writing or reviewing. Not for nginx as a static file server and not for Kubernetes Ingress.
---

# Traefik

Traefik watches the Docker socket and builds routes from container labels. There
is no config file listing your services — the containers describe themselves.

## The minimum that works

```yaml
services:
  traefik:
    image: traefik:v3.5
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--api.dashboard=true"
    ports: ["80:80", "8080:8080"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)"
      - "traefik.http.routers.whoami.entrypoints=web"
      - "traefik.http.services.whoami.loadbalancer.server.port=80"
```

`curl -H 'Host: whoami.localhost' localhost` should answer. `*.localhost`
resolves to 127.0.0.1 in Chrome and Firefox without touching `/etc/hosts`, which
makes it the fastest way to test.

## Four things a route needs — miss one and you get 404

1. **`traefik.enable=true`** on the container (required because of
   `exposedbydefault=false`, which you always want — otherwise Traefik invents
   routers for Postgres too).
2. **A router with a rule**: `Host(...)`, `PathPrefix(...)`, or both joined with
   `&&`.
3. **A service port**, when the container exposes anything other than exactly one
   port: `traefik.http.services.<name>.loadbalancer.server.port=8000`. Traefik
   guesses only when there is one `EXPOSE`d port, and guesses wrong otherwise.
4. **A shared network.** Traefik connects to the container directly, so both must
   be on the same Docker network. This is the most common cause of `502 Bad
   Gateway` with a route that otherwise looks correct.

Label grammar in full: `references/labels.md`.

## Security defaults

- **`:ro` on the Docker socket.** Write access to it is root on the host. Many
  published examples, including `awesome-compose/traefik-golang`, omit it.
- **`exposedbydefault=false`**, then opt in per container.
- **The dashboard is not authenticated.** `--api.insecure=true` on port 8080 is
  for a local machine only; in production route it through a `Host()` rule with
  basic auth (`references/middlewares.md`).

## Shared proxy on a dev machine (the default here)

One Traefik holds :80/:443 for every project; each project's compose file joins
an external network instead of running its own proxy.

```bash
docker network inspect traefik >/dev/null 2>&1 || docker network create traefik
```

```yaml
networks:
  traefik:
    external: true
services:
  api:
    networks: [traefik, default]
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=traefik"
      - "traefik.http.routers.api.rule=Host(`api.myapp.localhost`)"
```

- `external: true` means Compose will not create it — if it's missing, `up` fails
  with `network traefik declared as external, but could not be found`. Create it
  first (idempotent command above; as a Taskfile task, see `devops:taskfile`).
- **`traefik.docker.network=traefik`** is required as soon as a container is on
  more than one network. Otherwise Traefik may pick the project's default network,
  which it isn't on, and every request 502s.
- Keep an internal `default` network for app↔database traffic so the database is
  not on the proxy network at all.
- Alternative: Traefik as a service inside a single project's compose file
  (as `awesome-compose/traefik-golang` does). Simpler, self-contained, but only
  one project can hold port 80 at a time.

## Debugging a route

1. `docker compose logs traefik` — configuration errors are logged at startup.
2. The dashboard (`http://localhost:8080/dashboard/`) lists the routers,
   services and middlewares Traefik actually built. A router missing there is a
   label problem; a router present but red is a network or port problem.
3. `docker inspect <container> --format '{{json .Config.Labels}}' | jq` — see
   the labels as Docker stored them, after any `$`-escaping.
4. 404 → no router matched (rule, entrypoint, or `enable`). 502 → matched but
   Traefik cannot reach the container (network, port, or the app isn't listening
   on 0.0.0.0).

## Then

- Label grammar, rules, TCP/UDP routers → `references/labels.md`
- Local https with mkcert, and trusting the CA → `references/tls-local.md`
- Redirects, auth, prefixes, headers, Let's Encrypt → `references/middlewares.md`
- The container behind the route won't start → `devops:docker`

Sources: [Traefik docs](https://doc.traefik.io/traefik/),
[JensKnipper/traefik-examples](https://github.com/JensKnipper/traefik-examples).
