# Study Notes — Circuit Breaker, Retry & Bulkhead Patterns

## Definitions

- **Circuit Breaker**: A stateful proxy around a remote call that monitors failure rate and, once a threshold is crossed, stops sending requests to a struggling dependency ("fails fast") until the dependency has had time to recover.
- **Retry**: Automatically re-attempting a failed operation, typically with a delay strategy, to ride out transient/short-lived failures without surfacing them to the caller.
- **Bulkhead**: Partitioning resources (thread pools, connection pools, semaphores) per dependency or workload so that exhaustion in one partition cannot starve the others — named after a ship's watertight compartments.
- **Cascading failure**: A failure in one component (e.g., a slow dependency) that propagates upward by exhausting shared resources (threads, connections, queues) in the calling components.
- **Idempotency**: A property of an operation where applying it multiple times has the same effect as applying it once — a prerequisite for safe retries of non-read operations.
- **Thundering herd**: Many clients failing and retrying at the same synchronized moment, amplifying load on an already-struggling system.

## Circuit Breaker State Table

| State | Trigger to enter | Behavior while in state |
|---|---|---|
| Closed | Initial state / breaker reset after Half-Open success | Requests pass through normally; failures are counted over a rolling window (count or time-based) |
| Open | Failure rate (or consecutive failures) exceeds configured threshold while Closed | All calls fail immediately without contacting the dependency; starts a timeout ("sleep window") timer |
| Half-Open | Open-state timeout expires | A limited number of probe/test requests are allowed through to check recovery |
| → Closed | Probe requests in Half-Open meet the success threshold | Breaker resets failure counters, resumes normal traffic |
| → Open | Any (or enough) probe requests fail in Half-Open | Breaker re-opens and restarts the timeout timer |

Key tuning parameters: failure-rate threshold, rolling window size, open-state timeout duration, half-open probe count, half-open success threshold.

## Retry Strategy Comparison

| Strategy | Delay pattern | Pros | Cons |
|---|---|---|---|
| Fixed delay | Same wait time between every attempt (e.g., always 500ms) | Simple to implement and reason about | Doesn't back off under sustained failure; can contribute to thundering herd |
| Exponential backoff | Delay doubles (or grows exponentially) each attempt (100ms, 200ms, 400ms, 800ms…) up to a cap | Gives a recovering/overloaded dependency progressively more breathing room | If many clients fail simultaneously, they retry in synchronized waves |
| Exponential backoff + jitter | Random delay within (or around) the exponential window (e.g., AWS "full jitter": random(0, backoff)) | Spreads retries out over time, avoiding synchronized thundering herd | Slightly less predictable timing per client, but strictly better for system-wide load |

Other retry essentials:
- Cap total attempts and/or use a **retry budget** (e.g., retries ≤ X% of total request volume).
- Retries are only safe on **idempotent** operations, or non-idempotent ones protected by an **idempotency key** on the server.
- Never retry against an **Open** circuit breaker — let the breaker's fail-fast behavior stand.
- Distinguish retryable errors (timeouts, 5xx, throttling) from non-retryable ones (4xx validation errors, auth failures).

## Bulkhead Isolation Techniques

- **Thread pool isolation** — each downstream dependency gets its own bounded executor/thread pool so its exhaustion doesn't starve threads needed for other calls.
- **Connection pool isolation** — separate, capped DB/HTTP connection pools per downstream target.
- **Semaphore isolation** — a lightweight concurrency counter limiting in-flight calls to a dependency without the overhead of a dedicated thread pool (useful at very high throughput).
- **Separate service instances / node pools** — isolating workload classes or tenants at the infrastructure level so one noisy consumer can't consume all shared capacity.
- **Queue isolation** — dedicated, bounded request queues per dependency instead of one shared unbounded queue.

## Interview Revision — Quick Bullets

- Circuit breaker = fail fast + auto-recovery probing; three states: Closed, Open, Half-Open.
- Retry = handle transient failures; must use exponential backoff + jitter + retry budget + idempotency.
- Bulkhead = resource isolation per dependency; prevents one failure domain from starving others.
- All three combine: bulkhead limits concurrency → circuit breaker avoids wasted calls to a known-bad dependency → retry (with backoff/jitter) handles the remaining transient blips, but never retries against an Open breaker.
- Real implementations: resilience4j (Java, successor to Hystrix), AWS SDK built-in retry/backoff, Istio outlier detection/circuit breaking at the service mesh layer.
