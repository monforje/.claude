# Profiles, scenarios, and test data

## Contents

- [Which profile answers which question](#which-profile-answers-which-question)
- [Deriving the scenario mix from real traffic](#deriving-the-scenario-mix-from-real-traffic)
- [Think time and pacing](#think-time-and-pacing)
- [Warm-up](#warm-up)
- [Test data](#test-data)
- [Sessions, tokens, and correlation](#sessions-tokens-and-correlation)
- [When the environment is smaller than production](#when-the-environment-is-smaller-than-production)

## Which profile answers which question

| Profile | Shape | Answers |
| --- | --- | --- |
| **Smoke** | 1–5 VUs, 1–2 min | Does the script work? Right payloads, valid auth, checks asserting on real content. Always run this first; it costs a minute and saves whole invalid runs. |
| **Load (average)** | Ramp to expected peak, hold 15–60 min | Does the system meet its SLO at the load we actually expect? This is the run that produces the pass/fail verdict. |
| **Stress** | Ramp to 1.5–3× peak | What happens above expectations — graceful degradation or collapse? Does it recover when load drops? |
| **Breakpoint / capacity** | Ramp indefinitely until thresholds break | Where is the knee, and what saturates there? Use an abort-on-fail threshold so it stops instead of pounding a dead system. |
| **Soak / endurance** | Moderate load, 2–24 h | Leaks, unbounded caches, connection creep, log disks filling, tables growing, token expiry. Nothing shorter than an hour finds these, and they are the failures that take production down on day three. |
| **Spike** | Idle → peak in seconds, hold briefly, drop | Push notification, TV ad, cron stampede. Two findings: does it survive the jump, and does it *recover* (autoscaler lag, cold caches, retry storm, thundering herd on cache expiry). |
| **Scalability / step** | Steady steps at rising rates, measured per step | Builds the capacity curve — throughput and latency per level. What you use for planning ("we need 3 more instances for 2× traffic") rather than pass/fail. |

Pick from the question, and say which profile produced every number reported.
A p99 from a stress run and a p99 from an average-load run are different facts.

## Deriving the scenario mix from real traffic

The mix is the part most load tests get wrong, and it is the part with a factual
answer sitting in the access log.

Route distribution, with ids normalized so the routes actually group:

```bash
awk '{print $7}' access.log \
  | sed -E 's#/[0-9]+#/:id#g; s#/[0-9a-f-]{36}#/:uuid#g; s#\?.*##' \
  | sort | uniq -c | sort -rn | head -20
```

Peak arrival rate — per-minute counts, so a busy minute is not averaged away by
a quiet hour:

```bash
awk '{print $4}' access.log | cut -d: -f2,3 | uniq -c | sort -rn | head
# highest count / 60 = peak RPS
```

Also worth pulling before writing the script: the read/write ratio (a read-only
mix is comfortably optimistic), the authenticated share, payload sizes, and how
much traffic is served from cache or CDN today (load that never reaches the
origin in production should not reach it in the test either).

Then weight the scenarios to match. Where the log is unavailable — a new feature,
no access to production — say so and state the assumed mix as an assumption in
the report. An assumption a reader can challenge is fine; an invented mix
presented as measurement is not.

Keep the write paths in. Writes take locks, invalidate caches, produce queue
work, and fight the reads for the same pool — most contention only appears in a
mix.

## Think time and pacing

Real users pause between actions. Back-to-back requests from every VU is a stress
pattern: it overstates concurrency per user, understates cache hit rates, and
produces a bottleneck that does not exist in production.

Two different things, both useful:

- **Think time** — a pause between steps *within* a scenario, randomized
  (`sleep(1 + Math.random() * 2)`). Randomized, because identical sleeps make VUs
  march in lockstep and arrive in synchronized waves.
- **Pacing** — holding a fixed iteration period so each user performs, say, one
  checkout per minute regardless of how long it takes. With an arrival-rate
  executor the pacing is handled by the model; with a VU-based one you have to
  add it, or a slowing system silently reduces its own load.

Think time is also how a VU count relates to a user count: 200 VUs with 5 s of
think time per 250 ms request represent roughly 4000 concurrently browsing users.
Reporting "200 VUs" without the think time tells the reader nothing about how
many users that was.

## Warm-up

What is cold at the start of a run, and none of it is capacity:

- JIT-compiled runtimes are interpreting until a method is hot (the JVM in
  particular needs thousands of invocations, and tiered compilation takes
  minutes under low load).
- Connection pools open lazily — the first N requests each pay a connect.
- The database buffer pool and OS page cache are empty, so early queries read
  from disk.
- Application caches, CDN edges, and TLS session caches are empty.
- Autoscalers have not reacted yet, so the run starts on fewer instances than the
  steady state would have.

Run a warm-up phase at modest load and **exclude that window from the reported
numbers** — a tagged phase with thresholds scoped to the steady phase, as in
`k6.md`. How long is measurable rather than guessable: warm-up is over when
latency stops trending down, which is typically 1–5 minutes and longer on the
JVM. If the report says "p95 = 340 ms" and the first two minutes are inside it,
the number is neither the cold truth nor the warm one.

The exception worth remembering: if the question is *"what do users get after a
deploy"*, then cold start is the subject, and you measure the cold window on
purpose. Say which one you did.

## Test data

**Volume and shape.** A table with a thousand rows answers everything from
memory, and the query planner picks plans that will not survive production. What
has to be realistic is not the values but the shape: row counts within an order
of magnitude, cardinality and skew (one customer with 40% of the orders behaves
nothing like an even spread), index selectivity, text and blob sizes.

**Anonymization.** Production data in a test environment means real names,
emails, cards, and tokens somewhere with weaker access control, unencrypted
volumes, and logs nobody reads — and any live side effect in the run mails,
charges, or texts those people. Options in order of preference:

1. Generate synthetic data matching the shape. Safest, and reproducible.
2. Mask deterministically, preserving format, length, and uniqueness — same input
   maps to the same output so joins and constraints survive.
3. Drop the columns the test does not need at all, rather than masking them.

Whichever you choose, neutralize outbound side effects in that environment
(mail catcher, stubbed gateways) — anonymized data still hits real endpoints if
the code is pointed at them.

**Uniqueness — both ways.** Two opposite mistakes, symmetrical in effect:

- Every VU reading the *same* row: served entirely from cache, giving latency
  that no production traffic will reproduce.
- Every VU writing the *same* row: total lock contention, giving a bottleneck
  production does not have.

Spread reads across a realistic key distribution (with the skew real traffic
has — Zipf, not uniform, for anything user-facing), and spread writes across
distinct keys unless contention is the thing under test.

**Data that runs out.** One-time coupons, unique emails, single-use inventory: a
run consuming these fails halfway through and the errors look like a system
defect. Derive values from the iteration number
(`user_${exec.scenario.iterationInTest}@test.local`) so they are unique by
construction and the run is repeatable.

**Cleanup.** A load test writes a lot. Tag everything it creates — a dedicated
tenant, an id prefix, a marker column — so it can be deleted afterwards, and
delete it. Otherwise the next run starts against a database the previous run
bloated, and nobody can compare the two.

## Sessions, tokens, and correlation

Hardcoding a token that was valid when the script was written produces a run
where everything 401s and latency looks superb. Extract what the flow needs from
the responses — token, session cookie, CSRF value, created ids — and pass it
along, exactly as the real client does.

Authenticate in `setup()` where the token can be shared, and re-authenticate
inside a long soak run before expiry. But if login is part of the user journey,
keep it in the scenario with realistic weight: it is often the most expensive
endpoint in the system (password hashing is deliberately slow) and leaving it out
flatters the results.

## When the environment is smaller than production

Scaling down is normal. What matters is knowing which conclusions survive it.

**Survives:** relative comparisons between two commits on the same environment
(the most valuable load-test output there is); which resource saturates first,
usually; functional defects that only appear under concurrency — lost updates,
double charges, deadlocks, duplicated side effects; hard configuration limits
that are the same in both environments.

**Does not survive:** absolute capacity numbers; anything involving the network,
CDN, or geography if the generator sits next to the app; database behaviour when
data volume differs enough to flip a query plan; autoscaling and noisy-neighbour
effects; anything about a component that is stubbed in test and real in
production.

**If you must extrapolate**, scale by the narrowest shared resource rather than
instance count, and treat the result as an estimate with a stated basis
("staging holds 120 RPS on 1 app instance against a 4 GB database; production has
6 instances and a 64 GB database, so 700 RPS is a floor, not a prediction").
Extrapolation breaks completely across a threshold — a working set that fits in
cache in one environment and not the other, or an index that is used in one plan
and not the other — so say where you think the thresholds are.
