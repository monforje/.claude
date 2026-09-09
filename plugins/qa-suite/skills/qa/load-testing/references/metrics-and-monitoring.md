# Metrics, and where the bottleneck actually shows up

## Contents

- [The queueing intuition](#the-queueing-intuition)
- [Latency: percentiles and how to not ruin them](#latency-percentiles-and-how-to-not-ruin-them)
- [Open vs closed, and coordinated omission](#open-vs-closed-and-coordinated-omission)
- [Little's law, for sanity checks](#littles-law-for-sanity-checks)
- [Saturation signals per tier](#saturation-signals-per-tier)
- [Is it the generator?](#is-it-the-generator)
- [Reading a run](#reading-a-run)

## The queueing intuition

One number explains most of what a load test shows: **wait time grows as
1/(1−utilization)**. At 50% utilization a request waits about as long as it takes
to serve. At 80%, four times. At 90%, nine. At 95%, nineteen.

That is why latency looks flat and then explodes rather than degrading in
proportion, and why "CPU is only at 85%, there is headroom" is wrong — 85% of a
resource is already several times the service time in queueing. It is also why
**saturation, not utilization, is the useful signal**: the queue in front of a
resource reacts before the resource looks busy.

## Latency: percentiles and how to not ruin them

Report p50, p95, p99, and (for anything user-facing at scale) max. Never lead
with the mean: it is dominated by the bulk and blind to the tail, and the tail is
what generates retries, timeouts, and support tickets.

Two mistakes destroy percentile data:

**Averaging percentiles.** The mean of two instances' p99s is not the p99, and
neither is the average of per-minute p99s over an hour. Percentiles have to come
from the merged distribution — a histogram (Prometheus `histogram_quantile` over
summed buckets) or the raw samples. If your dashboard shows a p99 per pod and
someone averages them, the number is fiction.

**Buckets too coarse.** `histogram_quantile` interpolates inside a bucket, so if
your highest bucket is 1 s and real latency is 4 s, the p99 reads as ~1 s
forever. Check the bucket layout covers the range you expect under load, which is
much wider than the idle range.

One more thing worth stating in a report: **a page is slower than its slowest
percentile suggests.** If rendering a screen makes 30 backend calls, the chance
that none of them lands in the p99 is `0.99³⁰ ≈ 74%` — so a quarter of page
loads contain a p99 request. Tail latency compounds with fan-out.

## Open vs closed, and coordinated omission

A **closed model** (a fixed number of virtual users, each waiting for its
response before sending the next) has a property that quietly hides the failure
you are hunting: when the system slows, each user sends less often, so the
offered load *drops*. The system throttles its own load test. Latency looks bad
but bounded, and the run never shows what happens when arrivals keep coming.

Real traffic is **open**: users arrive at a rate that does not care how the
system feels. This is why capacity questions want an arrival-rate model
(`constant-arrival-rate` / `ramping-arrival-rate` in k6, a fixed rate in vegeta).

The name for what the closed model does to your data is **coordinated
omission**: the requests that would have been slow were never sent, so they are
missing from the distribution — a p99 computed from what was sent understates
reality, sometimes by an order of magnitude.

A closed model is still the right choice when the client really is a fixed pool:
a worker fleet with N threads, a partner limited to N connections, a mobile app
where each device has one in-flight request. Model what exists.

Detecting the problem mid-run: with an arrival-rate executor, active VUs climbing
means the system is slowing (that is the model absorbing it correctly), while
`dropped_iterations` above zero means the generator failed to offer the rate — at
which point the tested rate is not the reported rate.

## Little's law, for sanity checks

`concurrency = throughput × latency`

At 400 RPS with 250 ms average latency, about 100 requests are in flight, so the
generator needs at least ~100 VUs, and the server needs ~100 concurrent slots
(workers, threads, pool entries). Three uses:

- Sizing `preAllocatedVUs` before a run.
- Spotting an impossible result: 400 RPS reported by 20 VUs at 250 ms latency
  cannot be true; something is not measuring what you think.
- Finding a hard ceiling: if the app tier has 40 worker slots, then at 250 ms per
  request the arithmetic caps it at 160 RPS regardless of CPU. Many "mysterious"
  plateaus are just this.

## Saturation signals per tier

Utilization tells you a resource is busy. These tell you something is *waiting*,
which is where the answer usually is.

| Tier | Watch | Bottleneck when |
| --- | --- | --- |
| **Host** | CPU (per core), CPU steal, load average vs core count, `iowait`, memory + swap, NIC throughput and packet drops, conntrack table | Steal >5% means the neighbour, not you. `iowait` high with low CPU = disk. A full conntrack table drops connections with nothing in the app logs. |
| **Reverse proxy / LB** | Active connections vs `worker_connections`, upstream queue, 502/504 rate, keepalive reuse, open FDs vs `ulimit` | 502/504 while the app tier is idle means the proxy or its limits, not the app |
| **App runtime** | Worker/thread pool: busy vs max, queue depth, time queued. Go: goroutines, GC pause share. JVM: heap after GC, GC time %, pool queues. Node: event-loop lag. Python/PHP: gunicorn workers busy, `pm.max_children` reached | Queue time dominating service time is the classic "the app is slow" that is really "the app has no free worker". `pm.max_children reached` in the PHP-FPM log is a finished investigation. |
| **Connection pool** | Pool size, active, waiting, wait duration, timeouts | The most common bottleneck in web systems, and invisible without this metric: threads wait on the pool while CPU sits at 30% and the DB looks bored |
| **Database (Postgres)** | `pg_stat_activity` by state (`active`, `idle in transaction`, waiting), `pg_locks` waits, `pg_stat_statements` by total time, checkpoint frequency, cache hit ratio, replication lag | `idle in transaction` climbing is a transaction held open across an external call. Lock waits mean a hot row, not a slow query. |
| **Database (MySQL)** | `Threads_running`, `Innodb_row_lock_waits` and average wait, buffer pool hit rate, slow log | `Threads_running` far above core count = internal contention; adding load makes it worse |
| **Cache (Redis)** | Ops/sec, hit ratio, evictions, `blocked_clients`, slowlog, single-core CPU | Redis is effectively single-threaded per instance: one core at 100% is the ceiling even on a 32-core box. Evictions rising means the working set outgrew memory and the DB is about to feel it. |
| **Queue / workers** | Depth trend, consumer lag, oldest-message age, redelivery and DLQ rate | Depth rising monotonically = consumers are below producer rate; the system is not keeping up even though the API still answers fast |
| **Outbound dependencies** | Per-dependency latency and error rate, retry counts, circuit-breaker state | Retries multiply load; a retry storm shows as throughput falling while load rises |

Where these come from: `node_exporter` and `cAdvisor` for hosts and containers,
the runtime's own exporter or APM for the app tier, the database's exporter or
its stats views directly, Redis `INFO`, the broker's own metrics, and the
application's request metrics. If a tier has nothing, `docker stats`, `top`, and
a couple of psql queries in a loop beat guessing — take those readings at idle
first so the loaded numbers have a comparison.

## Is it the generator?

Before believing any bad result, clear the injector:

- Generator CPU under ~70% (a saturated injector reports its own latency).
- `dropped_iterations` / dropped-request counters at zero.
- Ephemeral ports and file descriptors not exhausted (`ss -s`, `ulimit -n`) —
  short-lived connections at high rate eat the port range, and the symptom is
  connection errors that look like the server refusing.
- DNS resolved once, not per request.
- Client-side connection setup flat: rising `http_req_blocked` or
  `http_req_connecting` at steady state is a client problem.
- The generator is not sharing a host with the system under test. Co-located, it
  competes for the same CPU and the run measures the fight.

The decisive check: add a second generator machine. If aggregate throughput goes
up, the first one was the limit and every previous number was about it.

## Reading a run

Plot arrival rate, completed throughput, latency percentiles, error rate, and
the saturation signals on one time axis, then read the shape:

| Shape | Meaning |
| --- | --- |
| Throughput tracks arrival rate, latency flat | Below the knee; there is headroom |
| Throughput flattens, latency climbs steeply | The knee — capacity found. Whatever is saturated here is the bottleneck |
| Latency climbs while throughput is flat and no resource is above 60% | A serialization point: a lock, a pool, a single-threaded component, an external call |
| Throughput *falls* as arrival rate rises | Retries, GC, or thrashing are consuming the system. It needs admission control, not hardware |
| Errors spike while latency *improves* | Failing fast. Check what the errors are before celebrating the latency |
| Latency creeps up over hours at constant load | A leak, an unbounded cache, growing tables, or disk filling — the soak-test finding |
| A step change at a round number | A configured limit: pool size, `max_connections`, a rate limiter, an autoscaler threshold |

Once the bottleneck is a specific piece of code rather than a tier, it stops
being a load-testing question — hand it to `qa-profiling`, which finds the line.
