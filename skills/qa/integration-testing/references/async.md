# Queues, workers, and eventual consistency

## First: try to make it synchronous

A test that never waits cannot flake on timing, so before reaching for a polling
loop, check whether the stack can run the work inline:

| Stack | Inline execution |
| --- | --- |
| Celery | `task_always_eager=True` (and `task_eager_propagates=True`, or failures vanish) |
| RQ / Sidekiq-likes | `Worker.work(burst=True)` — drain the queue, then assert |
| Kafka / SQS consumers | Call the handler loop once with a poll timeout instead of running the service |
| BullMQ | `Worker` with a manual `run()`, or process the job directly |
| Go workers | Call the consume function with a context that cancels after one batch |

The cost is honest and belongs in the report: inline execution skips
serialization, the broker, and retry policy. Keep at least one test that goes
through the real broker end to end, and use inline mode for the rest of the
behaviour.

## Where it must stay asynchronous: poll for the end state

```python
def wait_for(predicate, timeout=5.0, interval=0.05, what="condition"):
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        result = predicate()
        if result:
            return result
        time.sleep(interval)
    raise AssertionError(f"timed out after {timeout}s waiting for {what}")
```

```typescript
async function waitFor<T>(fn: () => Promise<T | null>, what: string, ms = 5000) {
  const deadline = Date.now() + ms;
  while (Date.now() < deadline) {
    const r = await fn();
    if (r) return r;
    await new Promise((res) => setTimeout(res, 50));
  }
  throw new Error(`timed out after ${ms}ms waiting for ${what}`);
}
```

Go: `require.Eventually(t, cond, 5*time.Second, 50*time.Millisecond, "waiting for %s", what)`.

Three things make the difference between this and a sleep:

- **Poll the observable end state.** "The message left the producer" proves
  nothing about the consumer. Poll the row, the cache entry, the outbound call —
  whatever a user would notice.
- **Name what you waited for.** A bare timeout at this level names none of the
  five moving parts that could have stalled; `no orders.status=shipped for order
  42 within 5s` names it.
- **Use a monotonic clock and a real timeout.** Not wall time, not an unbounded
  retry loop that hangs CI.

## Broker hygiene

- **Give each test its own consumer group and its own topic or queue** (suffix
  with a uuid). Sharing a group across tests means one test steals another's
  messages — the classic "passes alone, fails in the suite".
- **Create and delete topics inside the test**, or use a broker container per
  session with per-test naming. Leftover topics carry offsets that make the next
  run behave differently from the first.
- **Assert on idempotence, because delivery is at-least-once.** Deliver the same
  message twice and assert the end state is unchanged. If it is not, that is a
  finding whether or not the duplicate is likely today.
- **Test the failure path.** A handler that throws should end up wherever the
  system says it should — a retry, a dead-letter queue, a status column. Assert
  on that, not just on the happy path.
- **Ordering guarantees are per partition, not global.** A test that assumes
  global ordering passes on a single-partition test topic and fails in
  production; either pin the partition key deliberately or do not assert order.

## Retries, backoff, and timeouts

Real retry policies with exponential backoff make a test slow or flaky. Inject
the policy (attempts, delay) as configuration and use a tight one in tests —
three attempts at 10ms proves the same logic as three attempts at 30s. If the
policy cannot be injected, that is a design finding worth reporting.

## What never appears in these tests

Fixed `sleep`, wall-clock branching, unseeded randomness in message keys, or a
"wait until the CI machine is fast enough" comment. Each one is a future flake,
and a flaky integration suite trains the team to ignore a red build — which
costs more than the tests ever earned.
