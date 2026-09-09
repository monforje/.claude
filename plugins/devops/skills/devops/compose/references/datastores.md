# Postgres, MongoDB, Redis

Copy the block, change the credentials. The healthcheck is the part that makes
`depends_on: condition: service_healthy` work — don't drop it.

## Postgres

```yaml
  db:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret        # or POSTGRES_PASSWORD_FILE, see below
      POSTGRES_DB: app
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d:ro   # optional, first boot only
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
```

- **`pg_isready` without `-U` is a false positive.** It returns 0 while the
  server is still starting, and it checks the *server*, not that your role and
  database exist. Always pass `-U` and `-d`.
- **`/docker-entrypoint-initdb.d` runs only when the data directory is empty.**
  Edit an init script after the first `up` and nothing happens — because the
  volume already has data. `down -v` (destroys data) or run the SQL yourself.
  This surprises people constantly.
- **Never mount a bind mount at `/var/lib/postgresql/data` on macOS/Windows.**
  The filesystem translation breaks Postgres' locking. Named volume only.
- Upgrading `17` → `18` will not start on an old data directory: "database files
  are incompatible". Dump and restore, or use `pgautoupgrade`.
- Password from a file (keeps it out of `docker inspect`):
  `POSTGRES_PASSWORD_FILE: /run/secrets/db-password` + a `secrets:` entry.
- A one-shot migration job that must finish before the app:

```yaml
  migrate:
    build: .
    command: ["./migrate", "up"]
    depends_on: { db: { condition: service_healthy } }
    restart: "no"
  api:
    depends_on:
      migrate: { condition: service_completed_successfully }
```

## MongoDB — single-node replica set, not standalone

Transactions and change streams **require** a replica set. A standalone
container is the usual reason code that works against Atlas fails locally with
`Transaction numbers are only allowed on a replica set member`.

```yaml
  mongo:
    image: mongo:8
    restart: unless-stopped
    command: ["--replSet", "rs0", "--bind_ip_all"]
    ports: ["27017:27017"]
    volumes:
      - mongo-data:/data/db
    healthcheck:
      test: >
        mongosh --quiet --eval "try { rs.status().ok }
        catch { rs.initiate({_id:'rs0',members:[{_id:0,host:'mongo:27017'}]}).ok }"
      interval: 5s
      timeout: 5s
      retries: 30
      start_period: 5s
```

- The healthcheck doubles as the initiator: it fails until the set is up, then
  passes. That avoids a separate init container.
- `host` must be the **service name**, not `localhost` — the replica set config
  is handed to clients, and a client outside the container can't reach
  `localhost:27017` inside it.
- Connection string: `mongodb://mongo:27017/app?replicaSet=rs0&directConnection=true`.
  Without `directConnection=true` a driver connecting from the host tries to
  resolve the advertised member name `mongo` and hangs.
- With auth (`MONGO_INITDB_ROOT_USERNAME`/`_PASSWORD`) a replica set also needs a
  keyfile; for local development, run without auth and keep it off the network.

## Redis

```yaml
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
```

- **Default Redis persists nothing useful.** Without `--appendonly yes` and a
  volume at `/data`, a restart is an empty cache. Fine for a cache, fatal for a
  queue or session store — decide which it is.
- Password: `--requirepass ${REDIS_PASSWORD}` in `command`, and the healthcheck
  becomes `["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]`.
- Don't publish 6379 unless the host needs it. Unauthenticated Redis on a public
  interface is compromised in minutes.
- `redis:7-alpine` — the official image. `redislabs/redismod` (used by
  `awesome-compose/nginx-nodejs-redis`) is a module bundle, unpinned, and not
  what you want as a plain cache.

## Shared

```yaml
volumes:
  db-data:
  mongo-data:
  redis-data:
```

Named volumes live under `docker volume ls` as `<project>_<name>`. Change the
project name and you get a fresh, empty one — that is the "my data disappeared"
story almost every time.
