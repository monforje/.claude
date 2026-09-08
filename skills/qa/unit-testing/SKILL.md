---
name: unit-testing
description: Write, run, and debug unit tests that exercise one function, method, or class in isolation with its dependencies stubbed. Covers picking cases from the code's own branches, the mocking boundary, table-driven/parametrized tests, reading failures, and per-framework idioms for pytest, vitest/jest, and go test. Use this whenever unit tests are in play — adding tests for changed code, filling a coverage gap, fixing a failing or flaky unit test, or judging whether existing tests assert anything real — including when the user only says "add tests", "cover this function", or "why is this test red" without naming the level. Not for integration, E2E, or API tests that touch a real database, network, or browser.
---

# Unit Testing

A unit test pins one behaviour of one unit with everything around it replaced by
something the test controls. Its whole value is that a red test tells you where
to look without a debugger. Every rule below protects that property.

## How to write unit tests

**Test the contract, not the implementation.** The contract is what a caller can
observe: return values, raised errors, calls made to injected collaborators, and
state the caller can read back afterwards. Private helpers, call order inside the
function, and intermediate variables are free to change — asserting on them
produces tests that break during refactors while catching no bugs. That is the
main reason suites get abandoned.

**Get the cases from the code, not from imagination.** Read the unit and list its
branches: every `if`, every early return, every `catch`/`except`, every loop that
can run zero times, every boundary comparison (`<` vs `<=`), every nullable
input. That list *is* the test plan, and it is also how you know when you are
done. Then add the cases the code forgot — empty collection, zero, negative,
unicode, duplicate, expired — because those are where the defects live.

**Isolate at the seam the code already has.** A dependency you can replace is one
that arrives as an argument, a constructor parameter, or an interface. If the
unit reaches out and constructs its own database client or reads the clock
directly, you have found a design problem: report it rather than reaching for a
monkeypatch that welds the test to the implementation.

## Workflow

1. **Detect the stack.** `pyproject.toml`/`pytest.ini` → pytest; `package.json`
   (check `devDependencies` for vitest vs jest) → see the TS reference; `go.mod`
   → `go test`. Read the matching file in `references/` for idioms and commands.
2. **Read the unit.** List branches, boundaries, error paths, and injected
   dependencies before writing anything.
3. **Read one or two existing tests first.** Copy the repo's file layout, naming,
   fixture style, and assertion library. Consistency is what keeps tests alive.
4. **Write the happy path and run it.** One passing test proves the harness,
   imports, and fixtures work before you invest in twenty more.
5. **Add edge and error cases** from the branch list, running as you go.
6. **Make each new test fail once** — flip an assertion or comment out a line of
   the code under test. A test that has never been red is not known to test
   anything; this catches wrong mocks, unreached asserts, and typo'd names.
7. **Run the file, then the whole suite.** A green file with a broken suite means
   you leaked state.
8. **Triage every failure.** Code wrong → that is a finding; report it and leave
   the test red. Expectation wrong → fix the test. Never edit production code to
   turn a test green unless fixing the defect is the job.
9. **Report what is still uncovered and why** (needs a seam, belongs to
   integration, deliberately out of scope).

## Rules

- One reason to fail per test. Two behaviours in one test means a red run tells
  you less than it should.
- Name the test after the behaviour and the condition, not the function:
  `test_refund_rejected_when_order_already_settled`, not `test_refund_2`.
- Assert on values. "Did not throw" is not an assertion; neither is a snapshot
  nobody reads.
- Mock only what crosses a process boundary — network, disk, clock, randomness,
  subprocesses. Mocking your own pure code duplicates it in the test file and
  freezes today's structure.
- Never mock the unit under test, or any part of it.
- No sleeps, no wall clock, no unseeded randomness, no real network, no writes
  outside a temp dir. A flaky test is worse than no test: it trains people to
  ignore red.
- Tests must pass in any order and in isolation. Shared module-level mutable
  state is the usual culprit.
- Keep the arrange block short. If setup takes thirty lines, that is a message
  about the design — say so in the report.

## Patterns

- **Arrange / Act / Assert** (Given / When / Then). Three visually separate
  blocks; a reader should find the "act" line in one second.
- **Table-driven / parametrized tests** for one behaviour across many inputs.
  Give each row a name so failures identify themselves. Do not use a table when
  the rows need different assertions — that is two tests wearing a trenchcoat.
- **Fakes over mocks.** An in-memory implementation of the interface tests
  behaviour; a mock with five `assert_called_with` tests the call graph. Reach
  for mocks when the interaction *is* the contract (an email was sent).
- **Builders / factories for fixtures.** `make_order(status="settled")` keeps the
  test's one meaningful field visible instead of buried in twenty defaults.
- **Boundary triples.** For any threshold `n`, test `n-1`, `n`, `n+1`.
- **Characterization tests** for legacy code you must refactor: write tests that
  record what it does today, even if that behaviour looks wrong, then refactor
  against them. Flag the suspicious behaviour separately.
- **Property tests** where an invariant is easy to state (round-trips,
  idempotence, ordering) and enumerating cases is not.

## Tools

| Stack | Runner | Reference |
| --- | --- | --- |
| Python | pytest | `references/python.md` |
| TypeScript / JavaScript | vitest, jest | `references/typescript.md` |
| Go | `go test`, testify | `references/go.md` |

Use whatever the project already uses, with its config and its scripts
(`make test`, `npm test`, a `justfile` target) — inventing a second way to run
tests is how a suite stops being run in CI. Adding a new test dependency needs a
reason; say so in the report rather than adding it silently.

## How to verify the result

Before reporting done:

- [ ] Every branch listed in step 2 is either covered or explained.
- [ ] Each new test failed at least once for the right reason.
- [ ] The full suite passes (or the only red is a defect you are reporting).
- [ ] Tests pass when run in isolation and when the file order changes.
- [ ] Run the suite twice; identical results, no timing- or order-dependence.
- [ ] Coverage is checked on the changed lines, not on the global percentage.
- [ ] No production code was modified to make a test pass.
- [ ] The report names each defect as `[severity] file:line — what → fix`, lists
      the files touched, and gives the exact command to rerun the tests.
