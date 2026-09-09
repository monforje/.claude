---
name: load-testing
description: Design, run, and interpret load tests — the ones that answer how a system behaves under concurrent traffic, not whether one request is correct. Covers fixing the pass/fail criteria before generating load, k6 scripting (arrival-rate scenarios, stages, thresholds, checks), the metrics that decide the verdict (throughput, p95/p99 latency, error rate, resource utilization, saturation), reading infrastructure monitoring alongside the run, warm-up, realistic scenario mixes, and test data. Use this whenever load, stress, soak, spike, or capacity questions come up — "will it hold the sale", "why does p99 explode past 500 RPS", "find the bottleneck under load", "how many users can this take" — and whenever a k6/Locust/JMeter script needs writing, fixing, or its results interpreting. Not for finding which function is slow inside one request (that is profiling), and not for functional correctness.
---

# Load Testing

A load test answers one kind of question: **what happens to this system when many
things happen at once.** Not "is the answer right" — that is every other QA level
— but "at what arrival rate does the answer stop arriving in time, and what runs
out first".

That framing matters because a load test is easy to run and hard to make mean
anything. Pointing a generator at a URL and reading back "3400 RPS" takes two
minutes and tells you nothing: not whether 3400 is good, not whether the system
or the generator hit its limit, not what to fix. Everything below exists to turn
a number into a finding.

## Before generating any load

### The environment must be safe to hit, and you must be told so

**Generating load is a denial-of-service attack you are performing on your own
infrastructure.** That is not a metaphor — it is the same traffic, and the target
does not know the difference. So:

- **Never generate load against production** unless the user has explicitly said
  to, for that environment, in this conversation. "It's fine, it's just a test"
  from you is not authorization. Ask, and if the answer is no, say the
  production-load question stays unanswered rather than answering it by accident.
- **A shared staging environment has other people on it.** A load run makes it
  unusable for the duration and can poison it afterwards (exhausted quotas,
  rate-limited third parties, a queue backed up for hours). Getting a go-ahead
  is part of the setup, not politeness.
- **Watch what the system reaches out to.** Your load multiplies through every
  outbound call: payment sandboxes, SMS gateways, partner APIs, someone's
  free-tier webhook. Stub or disable outbound side effects, or you are load
  testing a stranger and possibly paying per request.

### The numbers are only as production-like as the environment

Latency measured against one container with 512 MB and an empty database
describes that container, not the system. Perfect fidelity is rare and rarely
affordable — the requirement is not that the environment matches production, but
that **every way it differs is written down next to the results**. Half the
instance count, no CDN, a database with 10k rows instead of 50M, a co-located
generator with no real network hop: each of those bends the answer in a knowable
direction, and a reader who knows about them can still use the report.
`references/scenarios.md` covers what scaling down does and does not preserve.

### Decide what "pass" means before you look at any output

A test without a criterion is a number generator, and numbers generated that way
get argued about instead of acted on. Fix these first:

| What | Example | Where it comes from |
| --- | --- | --- |
| Target load | 400 RPS sustained; 2× peak for the sale | Real traffic: access logs, APM, last year's peak |
| Latency budget | p95 < 300 ms, p99 < 1 s, per endpoint | The SLO, or the current production number if there is no SLO |
| Error budget | < 0.1% failed requests | The SLO, or "no worse than today" |
| Duration | 15 min steady; 4 h for the soak | Long enough for the question — see below |
| The question | "Does the new checkout hold last year's peak?" | The change, and the risk someone is worried about |

If none of that exists, derive the target from real traffic and **state the
derivation in the report** rather than inventing a round number. "Peak hour in
the access log is 220 RPS, tested at 440 as 2× headroom" is a defensible target.
"Tested at 1000 RPS" is trivia.

### Monitoring first, load second

This is the mistake that makes whole runs worthless: **a load test without
infrastructure monitoring produces a symptom and no cause.** You learn that p99
went to 4 seconds at 600 RPS. You cannot learn that it was the connection pool,
because nobody was watching the pool.

So before the first request goes out, know how you will read, for every tier:
CPU, memory, network, disk I/O, and the saturation signals — pool usage, queue
depth, thread pool, GC pauses, DB active connections and locks. Prove you can
read them *now*, while the system is idle, and take an idle baseline for
comparison. If a tier cannot be observed at all, say so up front: the run can
still happen, but the finding will be "the system saturates at ~600 RPS, cause
not localized because the database has no metrics exposed", and that is a
different, weaker deliverable.

`references/metrics-and-monitoring.md` has the per-tier signals and where they
come from.

## The metrics, and why each one is there

Five numbers, and the point is that **they are only meaningful together**:

- **Throughput (RPS)** — requests *completed* per second. Attempted RPS is the
  generator's plan; completed RPS is the system's answer.
- **Latency percentiles (p50 / p95 / p99)** — never the mean. A mean of 180 ms is
  compatible with 5% of users waiting eight seconds, and it is the 5% who
  complain, retry, and double your load. p50 says what a normal request feels
  like; p99 says what your worst-served real users get.
- **Error rate** — including what is not an HTTP status: timeouts, connection
  resets, and a `200` carrying an error body or an empty list. Without response
  assertions, a system that fails fast looks *faster*.
- **Resource utilization** — CPU, memory, network, disk on every tier. This is
  what turns "slow" into "CPU-bound on the app tier" or "not resource-bound at
  all, so it is a lock or a pool".
- **Saturation** — the queues in front of the resources: connection pool
  exhaustion, request queue depth, thread pool, GC pressure, DB lock waits.
  Saturation is usually where the answer is, because latency under load is
  mostly queueing, not work.

**Read them as one picture.** As you raise arrival rate, throughput climbs and
latency stays flat — until the knee, where throughput flattens (or falls) and
latency starts climbing steeply. That knee is the capacity, and whatever is
saturated at the knee is the bottleneck. Two shapes are worth recognizing on the
spot: latency rising while throughput is flat means a queue is filling somewhere,
and throughput *dropping* as load increases means the system is spending itself
on overhead — retry storms, GC, thrashing — and needs a limiter, not a bigger box.

## Common mistakes

These are the ones that silently invalidate a run, so they are worth checking
against every time rather than remembering to avoid.

**Testing without infrastructure monitoring.** Covered above. It is first because
it is the one that wastes the entire run.

**Using production data without anonymization.** A dump copied into a test
environment carries real names, emails, cards, and tokens into a place with
weaker access control, unencrypted volumes, and logs nobody watches — and the
test itself emails, charges, or texts those people if any side effect is live.
Anonymize, or generate synthetic data. What must be preserved is the *shape*
— row counts, cardinality, distribution skew, index selectivity — not the
values. See `references/scenarios.md`.

**Ignoring warm-up.** The first requests into a fresh process are slower for
reasons that have nothing to do with capacity: JIT still interpreting, empty
caches, a connection pool that lazily opens connections, a cold CDN, an
unprimed query planner. Averaged into the results they hide a real regression;
mistaken for the result they invent one. Run a warm-up phase and **exclude that
window from the numbers** — do not just hope it averages out.

**Hammering one endpoint instead of real scenarios.** `/api/products` at 5000 RPS
measures that endpoint's throughput and nothing about the system. Real users
arrive in mixes — browse, search, add to cart, check out — and the bottleneck is
almost always a *shared* resource that only contends under a mix: one connection
pool, one cache, one lock, one worker set. Weight the mix by what the access log
actually shows, and include the writes: read-only load tests are pleasantly
optimistic.

**Measuring the generator instead of the system.** A saturated injector reports
its own limits as the target's latency. Check the generator's own CPU, its
error/dropped counters, ephemeral ports, file descriptors, and DNS. Rule of
thumb: if generator CPU is above ~70% or its results change when you add a second
injector, the numbers are about your generator.

**Closed-model load hiding the problem.** With a fixed pool of virtual users, a
slowing system automatically receives *less* load — each user waits before
sending again, so the offered rate quietly drops exactly when you most want to
know what happens. Real traffic does not do this: users keep arriving. Prefer an
arrival-rate (open) model for capacity questions. This is
coordinated omission, and `references/metrics-and-monitoring.md` explains it and
when a closed model is still the right choice.

**Changing more than one thing between runs.** A comparison is only worth making
against a baseline that differs in one variable. New code *and* a rebuilt
database *and* a different profile produce a difference nobody can attribute.
Keep the profile, data, and environment fixed; change the code.

**An empty database.** A table with a thousand rows answers every query from
memory and every plan looks brilliant. Volume is part of the test.

## Workflow

1. **Frame the question and fix the criteria** (target load, latency budget,
   error budget, duration). Write them down before running anything.
2. **Confirm the target environment and get explicit permission** to load it.
   Note every way it differs from production.
3. **Set up observability and take an idle baseline.** Prove you can read each
   tier's utilization and saturation signals now.
4. **Derive the traffic shape from real data** — endpoint mix, weights, think
   time, payload sizes, the data set. `references/scenarios.md`.
5. **Write the script and smoke it at 1 VU.** A single-user run is where you find
   that auth expired, the payload is wrong, or the checks are asserting on a page
   that says "500". Every request must be asserted, or errors read as speed.
6. **Pick the profile for the question** — smoke, load, stress, breakpoint, soak,
   spike — with a warm-up phase in front and the steady window marked.
   `references/scenarios.md` has the shapes and what each one is for.
7. **Run, watching the system, not the terminal.** Rising latency with flat
   throughput, or a saturation signal pinning, is the run telling you where to
   look next.
8. **Localize the bottleneck to a tier** using utilization plus saturation. The
   deliverable is "the app tier's DB connection pool is exhausted at 480 RPS,
   p99 is queueing", not "it got slow".
9. **Hand off what is not yours.** Once the bottleneck is a specific piece of
   code, `qa-profiling` finds the line. A wrong result under concurrency — a lost
   update, a double charge — is a functional defect: report it, it is often the
   most valuable thing a load test finds.
10. **Report the numbers with the profile that produced them**, the verdict
    against the criteria from step 1, and every caveat about environment fidelity.

## Rules

- **Numbers without their run parameters are unusable.** Every reported figure
  carries the profile (arrival rate or VUs, ramp, duration), the environment, the
  data set, and the commit. Otherwise nobody can reproduce or compare it.
- **Assert every response.** An unasserted load test on a broken system reports
  excellent latency.
- **One run, one variable.**
- **Never report the mean as the headline.** Percentiles, and say which.
- **Do not average percentiles across time windows or instances.** The mean of
  two p99s is not a p99. Aggregate from the raw distribution.
- **A soak needs hours, not minutes.** Leaks, unbounded caches, log-disk
  exhaustion, and connection creep are invisible in a ten-minute run — they are
  precisely the failures that only appear on day three in production.
- **Fix the test and its scripts, never production code** to make a target pass
  — unless fixing the bottleneck was the job you were given.
- **Clean up.** Stop the generators, drain what you filled, and remove the test
  data you wrote. A load test leaves millions of rows behind if nobody thinks
  about it.
- **"It holds the target" is a real result.** Report it with the evidence and
  stop; do not escalate load until something breaks just to have a finding.

## Tooling

Default to **k6** unless the project already has something. It scripts in
JavaScript, models arrival rate natively, expresses pass/fail as thresholds in
the script itself, and exits non-zero in CI — which is most of what this skill
asks for, built in. `references/k6.md` has the recipes and the pitfalls, plus a
short table of when another tool is the better answer (Locust, JMeter, Gatling,
vegeta, ab).

**Use what the project already has.** A repo with a `locustfile.py` and a CI job
does not need a second tool; a second way to generate load is how load testing
stops being run. Extending an existing suite is routine work. Introducing a new
tool touches CI and everyone's machine — propose it, say what it buys, and wait
for an answer.

## References

| Topic | File |
| --- | --- |
| k6 recipes: executors, thresholds, checks, data, output, pitfalls; tool choice | `references/k6.md` |
| Metrics, percentile math, open vs closed model, per-tier saturation signals | `references/metrics-and-monitoring.md` |
| Load profiles, deriving the mix from traffic, warm-up, think time, test data | `references/scenarios.md` |

Read the one you need, not all three.

## How to verify the result

Before reporting done:

- [ ] The target load, latency budget, and error budget were written down before
      the run, and the report states where they came from.
- [ ] The user explicitly authorized loading this environment; nothing was run
      against production without that.
- [ ] Observability was verified while idle, and an idle baseline was captured.
- [ ] The script was smoked at 1 VU and every request is asserted — no run whose
      "speed" is a wall of unchecked errors.
- [ ] There was a warm-up phase, and it is excluded from the reported numbers.
- [ ] The load was a realistic scenario mix, including writes — not one endpoint,
      unless one endpoint was the question.
- [ ] The generator was not itself saturated (its CPU, errors, and ports were
      checked).
- [ ] Latency is reported as percentiles from the raw distribution, with
      throughput, error rate, utilization, and saturation for the same window.
- [ ] The bottleneck is localized to a tier and a resource, or the report says
      plainly that it could not be localized and why.
- [ ] Every reported number carries its profile, environment, data set, and
      commit; caveats about environment fidelity are listed.
- [ ] No production data was used unanonymized.
- [ ] Generators are stopped and the test data written during the run is cleaned
      up.
- [ ] The report names findings as `[severity] <where> — what → fix`, gives the
      exact command to rerun, and states the verdict against the criteria.
- [ ] If the level could not be covered at all — no safe environment, no
      permission, no observability — the report says
      `[major] load risk not covered — <reason> → <what would cover it>` and the
      verdict is not "success".
