# Databases

## Choosing the reset mechanism

The invariant is in SKILL.md: each test creates its own data and asserts only
about it. These are the ways to hold that invariant.

**Transaction rollback.** Open a transaction in the fixture, hand the connection
to the code, roll back at the end. Fastest and perfectly clean — but only when
the code under test does not commit for itself and does not open its own
connection. With SQLAlchemy, bind the session to an outer connection and use a
`SAVEPOINT` so the code's own `commit()` stays inside your transaction:

```python
@pytest.fixture
def session(engine):
    conn = engine.connect()
    trans = conn.begin()
    s = Session(bind=conn, join_transaction_mode="create_savepoint")
    yield s
    s.close(); trans.rollback(); conn.close()
```

Go: `sql.Tx` passed to the repository. Node: the same pattern with Prisma's
interactive transactions or a Knex transaction handle.

**Truncate between tests.** The default that always works, including when the
test drives the app over HTTP. One statement, once per test, in dependency order:

```sql
TRUNCATE TABLE orders, order_items, customers RESTART IDENTITY CASCADE;
```

Generate the table list from the catalog rather than hand-maintaining it —
`information_schema.tables` minus the migration bookkeeping table. Never
truncate the migrations table; re-running migrations per test is the slowest
possible choice.

**Unique data per test.** Every row the test creates carries a value unique to
the test (a tenant id, a uuid prefix). Required for parallel runs against one
database; costs you global assertions, so write `count(*) where tenant = ...`
rather than `count(*)`.

**Fresh schema per test.** A separate schema (`SET search_path`) or database per
test. Reserve it for tests of migrations and destructive DDL — it is an order of
magnitude slower than truncation.

## Migrations

Build the test schema the way production builds it: run the project's migration
command. Creating tables from ORM metadata (`Base.metadata.create_all`) tests a
schema that only exists in tests, and it hides exactly the drift you are looking
for.

Testing a migration itself is one of the strongest reasons to be at this level,
and it needs three assertions the migration author usually skips:

- It applies to a database that already holds representative data, not to an
  empty one. Backfills and `NOT NULL` additions fail only on real rows.
- The data after it is what was intended — check a few rows, not just that the
  column exists.
- The down migration, if the project keeps them, returns to a working state.

Apply migrations once per session, not once per test; combine with truncation.

## Dialect

Do not substitute SQLite for Postgres, or an embedded engine for the real one.
It catches typos and misses everything the level exists for: `ON CONFLICT`,
window functions, JSONB operators, array types, real type coercion, actual
constraint and locking behaviour. If the production database is Postgres 15, the
test database is Postgres 15.

## Assertions

**Read back through a fresh session.** After the code saves, the object you hold
may be answering from an identity map or a first-level cache, so asserting on it
proves only that you assigned a field. Expire or close the session and load the
row again — or check with a plain SQL query, which is immune to ORM behaviour.

**Assert on what the code was supposed to change, and on what it was not.** A
partial update test is only meaningful if it also shows the neighbouring columns
untouched.

## Traps worth a test

- Uniqueness and foreign-key constraints under a real concurrent insert, not in
  theory.
- Timestamps: database `now()` and application `now()` are different clocks and
  usually different time zones. Pick one and assert with a tolerance window.
- Sequences and identity columns after `RESTART IDENTITY` — anything that
  hard-codes id `1` will pass alone and fail in a suite.
- `SELECT ... FOR UPDATE` and lock ordering: two connections in one test are the
  only way to see a deadlock before production does.
