---
name: integration-testing
description: Write, run, and debug integration tests — the ones that need something actually running: a real database, a message queue, a cache, a second service. Covers using the harness the project already has (testcontainers, docker compose), test-data isolation, waiting for asynchronous work without sleeps, migrations, and stubbing third-party HTTP. Use this whenever a test requires something to be started — verifying a repository against a real DB, a producer/consumer round-trip, a schema migration, a cache invalidation — and whenever integration tests pass locally but fail in CI, leak data between runs, or flake. Not for unit tests with everything mocked (use unit-testing), not for the outward HTTP contract of an endpoint, not for full user journeys through the whole system.
---

# Integration Testing

An integration test checks that your code talks correctly to the things it leans
on — a database, a queue, a cache, another service. It buys you what no amount of
mocking can: the SQL dialect, the transaction that never commits, the migration
that fails on real data, the message whose schema drifted. It costs you a running
dependency, and everything running is a source of flakiness. Every rule below
exists to keep the first without paying the second.

## What belongs at this level

The test is mechanical: **if the test needs something started, it belongs here.**
A repository against a real Postgres, a producer and consumer round-tripping
through a real broker, a migration applied to a real schema, a cache invalidation
that only misbehaves against real Redis.

What does not belong here:

- Everything the code touches is replaced by something the test controls — that
  is a unit test, and the `unit-testing` skill covers it.
- The outward contract of an endpoint: status codes, response schema, auth,
  versioning, request validation. That question belongs to `qa-api`.
- A full user journey through the assembled system — `qa-e2e`.

The line against `qa-api` is the question, not the transport. An integration test
may well drive the app over HTTP because that is how the code really runs; it is
still an integration test if what it checks is "did the row land in the table,
and is it the right row". It is an API test if what it checks is "is 422 the
right status for this body".

## Start with the harness, not with the tests

Writing twenty tests and only then discovering there is nothing to run them
against is the most expensive possible ordering, and at this level it is the
likely one. So three things happen before the first assertion.

**Check that the runtime is actually available.** `docker info` answers it in a
second. No daemon, no permissions, a CI runner without docker-in-docker — say so
immediately and put the choice to the user: make it available, or drop the work
to the unit level with the integration risk reported as explicitly uncovered.
Never quietly fall back to fakes. A fake at this level is a unit test in a
costume, and reporting it as integration coverage is worse than reporting
nothing, because it retires the risk without testing it.

**Find the harness the project already has.** Look for `docker-compose*.yml`
(especially a `.test`/`.ci` variant), testcontainers in the dependency manifest,
a `make test-integration` / `just` / npm-script target, the CI workflow (it shows
how the suite really runs), and session-scoped fixtures in `conftest.py`,
`globalSetup`, or `TestMain`. Use exactly that. A suite that only goes green the
way one agent invented is a suite CI cannot run.

**If there is none, propose one and wait for an answer.** A harness is not a
private matter: it lands in CI, in onboarding, on every colleague's machine.
Show the smallest thing that works — a compose file or a testcontainers fixture —
say what it adds to the project, and write tests once the answer is yes. The
absence of a harness is itself a finding worth reporting either way.

**Extending a harness is not introducing one.** Adding Redis to a compose file
that already exists, or a testcontainers module next to the Postgres one, stays
inside the mechanism the project already chose — do it without ceremony.
Switching mechanisms, or adding a second way to bring the environment up, is the
conversation above. Two ways to start the environment is how a suite stops being
run.

## Never test against something you did not start

The connection string comes from the ephemeral environment the test brought up —
never from the project's `.env`, never from a config profile, never from an
environment variable that happens to be set on the machine.

This is the one place in this skill where a mistake destroys someone's data
instead of wasting time. Integration tests write, and then they truncate. A
"clean dev database" is regularly someone's working stand, a shared staging
schema is regularly the only copy of a reproduction someone spent a day on. If
the only reachable database is one you did not provision, that is a stop — report
it and ask, do not point the suite at it.

## Isolation: every test builds its own world

The invariant, and it is the whole of it: **each test creates the data it needs
and asserts only about that data; running order and parallelism change nothing.**
A test that passes only after another test ran is not a test, it is a ritual.

Choose the reset mechanism from how the test drives the system:

| Mechanism | Use when | Breaks when |
| --- | --- | --- |
| Roll back a transaction per test | The test calls in-process code that does not commit for itself | The code manages its own transactions, or the test goes over HTTP (a different connection is a different transaction) |
| Truncate/delete between tests | Anything else; the reliable default | Slow on large schemas; blocks parallel runs against one database |
| Unique data per test (own tenant, prefix, or id) | Tests run in parallel | Forbids global assertions like "the table has exactly one row" |
| Fresh schema or database per test | Migrations, destructive DDL | Slowest by a wide margin; reserve it |

And one prohibition: **no shared seed fixture that fills the database for the
whole run.** It is convenient exactly once and then becomes the reason nobody
runs the suite — every test silently depends on rows it did not create, and any
edit to the seed breaks tests that have nothing to do with it.

`references/databases.md` has the per-mechanism patterns, plus migrations and
dialect traps.

## Waiting for asynchronous work

You put a message on a queue; a worker handles it eventually. `sleep(2)` makes
the test either flaky or two seconds slower than it needs to be — usually both.

**First, try to remove the asynchrony instead of waiting it out.** Most stacks
can run the handler inline in tests: an eager execution mode, a consumer polled
once and drained synchronously, a scheduler stepped by hand. A test that does not
wait cannot flake on timing. The tradeoff is honest and worth stating in the
report: inline execution does not exercise the real dispatch path, so keep at
least one test that goes through the broker for real.

**Where that is impossible, poll for the observable end state with a time
budget** — and make the timeout message say what it was waiting for
("no `orders.status = shipped` for order 42 within 5s"), because a bare
`TimeoutError` at this level tells the reader nothing about which of five moving
parts stalled. Poll on the end state, not on an intermediate one: a message
having left the producer proves nothing about the consumer.

Fixed sleeps are out, and so is anything that reads the wall clock to decide.
`references/async.md` has the polling patterns, consumer-group hygiene, and what
at-least-once delivery obliges you to assert.

## Third-party services

Your own dependencies run for real. Somebody else's do not: stub them at the HTTP
boundary (respx, nock/msw, wiremock, httpmock — see `references/harness.md`).
Real vendors in CI mean flakes, secrets, and rate limits.

Be honest about what that buys: a stub written from documentation verifies your
understanding of the contract, never the contract. Say so in the report. If the
vendor offers a sandbox, a separate, clearly marked suite that runs outside the
default pipeline is a reasonable thing to *propose* — it is not something to add
unasked.

## Workflow

1. **Check the runtime and find the harness** (see above). Do not skip ahead;
   everything downstream depends on the answer.
2. **Read the change and locate the seams.** Which component now talks to which
   dependency, and what could go wrong in the conversation: a query, a
   transaction boundary, a migration, a serialization format, a retry.
3. **Read one or two existing integration tests.** Copy the project's fixtures,
   naming, markers, and the command CI uses. Consistency is what keeps a suite
   alive.
4. **Bring the environment up and prove it** with one trivial test that connects
   and reads back a row it wrote. Debugging a real failure through an environment
   you have not yet verified wastes hours.
5. **Write the happy path across the seam**, then the failures that only exist at
   this level: constraint violations, a dependency that is down, a timeout, a
   duplicate message, a concurrent writer.
6. **Make each new test fail once** — break the query, point it at the wrong
   table. A test that has never been red at this level is often testing that two
   `None`s are equal.
7. **Triage every failure.** Code wrong → a finding; report it and leave the test
   red. Expectation wrong → fix the test. Harness wrong (leaked data, missing
   wait, port collision) → fix the harness; that is a defect in the tests, and it
   is yours to fix.
8. **Verify per the checklist below** and tear down whatever you started.
9. **Report** findings, files touched, the exact command to rerun, and anything
   left uncovered with the reason.

## Rules

- Assert by reading back through a *fresh* session or connection. Asserting on
  the object you just saved often checks your ORM's identity map and nothing
  else.
- One seam per test. "Order is saved and the email is queued and the cache is
  invalidated" tells you nothing useful when it goes red.
- Fix tests and their harness, never production code — unless fixing the defect
  was the job you were given. Changes to `docker-compose`, CI config, or the
  migration history reach beyond the test suite and need the user's agreement.
- Keep tests runnable individually. `pytest path::test_name` on one integration
  test must pass on its own; if it needs a sibling to have run first, isolation is
  broken.
- No fixed sleeps, no wall-clock dependence, no test-ordering dependence, no data
  left behind after the run.
- Speed is a feature here: an integration suite that takes twenty minutes gets
  moved to nightly and then ignored. Prefer one environment for the whole session
  over one per test, and say so if the suite is getting slow.

## References

| Topic | File |
| --- | --- |
| Databases: reset mechanisms, transactions, migrations, dialect traps | `references/databases.md` |
| Queues, workers, eventual consistency, waiting without sleeps | `references/async.md` |
| Harness: testcontainers and compose per stack, Docker checks, HTTP stubs | `references/harness.md` |

Read the one that matches the dependency that changed, not all three.

## How to verify the result

Before reporting done:

- [ ] Every seam identified in step 2 is either covered or explained.
- [ ] Each new test failed at least once for the right reason.
- [ ] The suite passes **twice in a row without recreating the environment** —
      the second run works on whatever the first left behind. This is the check
      that catches leftover-data dependence, and it is the most valuable line
      here.
- [ ] The suite passes **from a completely fresh environment** — containers and
      volumes removed, migrations applied by the project's own command. This is
      what separates "it works" from "it works on the container I hand-patched
      three days ago".
- [ ] Every test passes when run on its own.
- [ ] No fixed sleeps, and every wait has a timeout with a message naming what it
      waited for.
- [ ] Nothing was run against a database, queue, or cache the test did not start.
- [ ] Whatever the run started is stopped; nothing was left listening.
- [ ] No production code was modified to make a test pass.
- [ ] The report names each defect as `[severity] file:line — what → fix`, lists
      the files touched, and gives the exact command to rerun the suite.
- [ ] If the level could not be covered at all — no runtime available, harness
      declined — the report says so as
      `[major] integration risk not covered — <reason> → <what would cover it>`,
      and the verdict is not "success".
