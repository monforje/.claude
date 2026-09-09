# Local https that the browser believes

Goal: `https://app.myproject.local` with no warning, no `--insecure`, and no
certificate files committed to the repository. Four steps.

## 1. Issue a certificate into a Docker volume

Keeping the certificate and its CA in a named volume — not in the repo — means
nothing secret is ever near git, and every project on the machine reuses one CA.

```bash
docker volume create myproject-certs

docker run --rm \
  -v myproject-certs:/certs -e CAROOT=/certs -e HOME=/certs \
  golang:1.25-alpine \
  go run filippo.io/mkcert@latest \
    -cert-file /certs/myproject.local.pem \
    -key-file  /certs/myproject.local-key.pem \
    myproject.local "*.myproject.local"
```

A wildcard plus the apex covers every subdomain you will add later, so this runs
once. `CAROOT=/certs` puts the root CA in the volume too — step 4 needs it.

## 2. Hand it to Traefik through the file provider

Traefik takes certificates from a dynamic configuration file, not from labels.
Write it into the same volume:

```yaml
# /certs/dynamic.yml
tls:
  certificates:
    - certFile: /certs/myproject.local.pem
      keyFile: /certs/myproject.local-key.pem

http:
  routers:
    myproject-https-redirect:
      rule: "HostRegexp(`^(.+\\.)?myproject\\.local$`)"
      entryPoints: [web]
      middlewares: [to-https]
      service: noop@internal
  middlewares:
    to-https:
      redirectScheme:
        scheme: https
        permanent: true
```

**Why the redirect lives here and not on the entrypoint.** The obvious
alternative is a global redirect
(`--entrypoints.web.http.redirections.entrypoint.to=websecure`). Don't, if the
machine hosts anything else: it redirects *every* http request on :80, including
container-to-container calls to other local hostnames you have no certificate
for. Webhooks between containers speak plain http and do not follow redirects,
so those break with no useful error. Scoping the redirect to your own domain
with `HostRegexp` leaves everything else alone.

Note the `\\.` — this is YAML, so the backslash needs escaping.

## 3. Run Traefik with the volume and the file provider

```bash
docker run -d --rm --name traefik \
  --network traefik \
  --network-alias myproject.local \
  --network-alias app.myproject.local \
  -p 80:80 -p 443:443 \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v myproject-certs:/certs:ro \
  traefik:v3.5 \
  --providers.docker=true \
  --providers.docker.exposedbydefault=false \
  --providers.file.filename=/certs/dynamic.yml \
  --entrypoints.web.address=:80 \
  --entrypoints.websecure.address=:443
```

**The `--network-alias` flags are the non-obvious part.** On the host,
`app.myproject.local` resolves through `/etc/hosts` to 127.0.0.1 and reaches
Traefik. Inside the Docker network there is no `/etc/hosts` entry, so a container
calling `https://app.myproject.local` gets NXDOMAIN. An alias makes the proxy
answer to the public name from inside the network too, so the same URL works in
both places — which is what a frontend dev server proxying to a backend, or one
service calling another by its public name, actually needs.

Services then attach with ordinary labels plus `tls=true`; no `certresolver`,
because the certificate comes from the file provider.

## 4. Trust the CA on the host

Two separate stores, and the second one is what people miss.

**System store** (curl, most language runtimes, Firefox on some setups):

```bash
docker run --rm -v myproject-certs:/certs alpine cat /certs/rootCA.pem > /tmp/rootCA.pem
sudo cp /tmp/rootCA.pem /usr/local/share/ca-certificates/myproject-local-ca.crt
sudo update-ca-certificates                    # Debian/Ubuntu
# Fedora/RHEL: sudo cp ... /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust
# macOS: sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain /tmp/rootCA.pem
```

**Chrome/Chromium on Linux does not read the system store.** It has its own NSS
database at `~/.pki/nssdb`, and only `certutil` writes to it. Rather than
installing `libnss3-tools` on the host, borrow it from a container:

```bash
mkdir -p "$HOME/.pki/nssdb"      # create it as your user first; see below
docker run --rm -v "$HOME/.pki:/pki" -v /tmp:/certs:ro alpine sh -c \
  'apk add --no-cache nss-tools >/dev/null &&
   certutil -d sql:/pki/nssdb -A -t C,, -n "myproject local CA" -i /certs/rootCA.pem &&
   chown -R '"$(id -u):$(id -g)"' /pki'
```

Restart Chrome afterwards — it reads the database at startup. Skipping this step
gives you a valid certificate that Chrome still marks "Not Secure", which reads
like a certificate problem and isn't.

Create `~/.pki/nssdb` **before** the bind mount: Docker creates a missing mount
path as root, and Chrome then cannot write to its own database.

**Hostnames.** Add them to `/etc/hosts`:
`127.0.0.1 myproject.local app.myproject.local`. A dev-only alternative that
needs no `/etc/hosts` and no CA at all is `*.localhost`, which browsers resolve
to 127.0.0.1 — but it gives you http only.

## Making it repeatable

All four steps are idempotent checks plus a command, which is exactly what
Taskfile `status:` is for — see `../../taskfile/references/docker-patterns.md`.
Step 4 is the only one that needs `sudo`, so guard it with a status check and it
won't prompt on an already-configured machine.

For a real domain use Let's Encrypt instead: `middlewares.md`.
