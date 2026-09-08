# Harness: bringing the environment up

## Step zero: is there a runtime at all

```bash
docker info >/dev/null 2>&1 && echo ok || echo "no docker"
```

Run this before writing anything. If it fails, stop and put the choice to the
user — make Docker available, or drop to the unit level and report the
integration risk as uncovered. A podman socket, a remote `DOCKER_HOST`, or a
CI runner with docker-in-docker all satisfy this; a fallback to fakes does not.

## Finding what the project already uses

Check, in this order, and use whatever you find:

```bash
ls docker-compose*.y*ml compose*.y*ml 2>/dev/null
grep -rEl "testcontainers" --include=pyproject.toml --include=requirements*.txt \
  --include=package.json --include=go.mod .
grep -rn "integration" Makefile justfile package.json 2>/dev/null
ls .github/workflows/*.y*ml .gitlab-ci.yml 2>/dev/null   # how CI really runs it
```

The CI config is the most reliable source: it is the one description of the
environment that has to be true.

## testcontainers

Preferred when the project already depends on it: the lifecycle lives in the
test code, so there is no separate "did you remember to start the stack" step,
and ports are random, so two runs never collide.

```python
# Python — one container per session, truncation per test
@pytest.fixture(scope="session")
def pg():
    with PostgresContainer("postgres:15") as c:
        yield c.get_connection_url()
```

```typescript
// TypeScript
const container = await new PostgreSqlContainer("postgres:15").start();
process.env.DATABASE_URL = container.getConnectionUri();
// afterAll: await container.stop();
```

```go
// Go
ctx := context.Background()
pg, err := postgres.Run(ctx, "postgres:15",
    postgres.WithDatabase("app"), testcontainers.WithWaitStrategy(
        wait.ForLog("ready to accept connections").WithOccurrence(2)))
defer testcontainers.CleanupContainer(t, pg)
```

Scope containers to the session, not to the test — a container per test turns a
30-second suite into a 10-minute one. Combine session-scoped containers with
per-test truncation (see `databases.md`).

## docker compose

Used when the project already has a compose file. Two rules:

- **Wait on a healthcheck, not on a sleep.** `depends_on: condition:
  service_healthy` with a real `healthcheck` (`pg_isready`, a broker probe).
  "Port is open" is not "ready to accept queries", and the gap between them is
  where the first test of every run fails.
- **Do not hard-code host ports** in the test compose file. A fixed `5432:5432`
  collides with the developer's own database and with a parallel CI job; publish
  an ephemeral port and read it back, or run the suite inside the compose
  network.

Reuse the project's file. If a variant is needed, extend the existing one
(`docker compose -f docker-compose.yml -f docker-compose.test.yml`) rather than
writing a parallel stack.

## Teardown

Stop what you started; leave alone what was already running when you arrived. If
the environment was up before the run, tearing it down destroys state that was
not yours to destroy — reuse it and say so in the report. Named volumes survive
`docker compose down` by design, which is why the "fresh environment" check in
the SKILL.md checklist uses `down -v`.

## Stubbing third-party HTTP

| Stack | Tool |
| --- | --- |
| Python | `respx` (httpx), `responses` (requests), `pytest-httpserver` for a real socket |
| TypeScript | `msw`, `nock` |
| Go | `httptest.NewServer`, `jarcoal/httpmock` |
| Any / cross-process | WireMock or MockServer in a container |

Prefer a real local server (`pytest-httpserver`, `httptest`, WireMock) over
in-process patching when the code under test runs in another process or uses a
client you do not control — patching only works inside your own interpreter.

Record what the stub asserts: a stub written from a vendor's documentation
verifies your reading of the contract, never the contract. Say that in the
report rather than letting a green suite imply more than it proved.

## CI notes

The suite must run in CI with the same command as locally. If it needs a
service the CI file does not declare, adding it there is a change beyond the
test suite — propose it, do not commit it silently. Mark integration tests so
they can be selected (`-m integration`, a build tag, a separate npm script):
someone will want the fast suite on every push and this one on merge.
