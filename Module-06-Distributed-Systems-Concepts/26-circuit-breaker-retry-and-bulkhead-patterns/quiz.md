# Practice & Interview Questions

**1. What are the three states of a Circuit Breaker, and what triggers each transition?**
Closed (normal, requests pass through and failures are counted), Open (failure rate/consecutive failures crossed a threshold, so all calls fail immediately without contacting the dependency), and Half-Open (the open-state timeout expired, so a limited number of probe requests are allowed through). From Half-Open, enough successful probes transition back to Closed, while any/enough failed probes send it back to Open.

**2. Why does a Circuit Breaker "fail fast" instead of just letting requests time out naturally?**
Letting every request run to its full timeout keeps threads, connections, and queue slots tied up on a call that's very likely to fail anyway. Failing fast immediately frees those resources for other work, which is exactly what prevents the caller's own resource pool from being exhausted and cascading the failure upward.

**3. Why is naive retry-on-failure dangerous during an outage?**
If a dependency is already struggling and every client immediately retries on failure, the retries add more load on top of an already-overloaded system, making the outage worse — this is the thundering herd effect. Without backoff, a retry storm can turn a partial degradation into a full outage.

**4. Explain exponential backoff with jitter and why jitter matters.**
Exponential backoff increases the delay between retry attempts exponentially (e.g., 100ms, 200ms, 400ms, 800ms) to give a struggling dependency time to recover. Jitter adds randomness to each delay so that many clients who failed at the same moment don't all retry at the exact same moment again — without jitter, backoff alone can still produce synchronized waves of retries.

**5. Why must retries be limited to idempotent operations, or otherwise protected?**
An idempotent operation produces the same result no matter how many times it's applied, so retrying it is safe (e.g., a GET or a PUT that sets an absolute value). A non-idempotent operation, like "charge this credit card" or "increment this counter," can cause duplicate side effects if retried blindly — this is typically solved by having the client send an idempotency key that the server uses to detect and safely deduplicate repeated requests.

**6. What is a retry budget, and why is it useful?**
A retry budget caps the number or percentage of retries allowed (e.g., retries must not exceed 10% of total request volume). It prevents retries from silently ballooning traffic during a widespread outage, keeping the system's overall load bounded even in worst-case failure scenarios.

**7. What is the Bulkhead pattern, and what real-world concept is it named after?**
The Bulkhead pattern isolates resources (thread pools, connection pools, semaphores) per dependency or workload so that exhaustion in one does not starve the others. It's named after a ship's watertight compartments (bulkheads), which contain flooding to one section instead of sinking the whole vessel.

**8. Name three concrete ways to implement bulkhead isolation.**
(1) Dedicated thread pools per downstream dependency, (2) separate, capped connection pools per downstream target, (3) semaphore-based concurrency limits for lightweight isolation without a full thread pool, and/or separate service instances or node pools to isolate tenants/workloads at the infrastructure level.

**9. A downstream payment service is slow but not fully down. Design the retry + circuit breaker behavior you'd want.**
Wrap the payment call in a bulkhead (dedicated thread pool/semaphore) so its slowness can't exhaust threads needed elsewhere. Wrap it in a circuit breaker tracking failure/timeout rate over a rolling window; once the threshold trips, the breaker opens and fails fast rather than continuing to queue requests against a struggling service. Retries should use exponential backoff with jitter and a small bounded attempt count, applied only while the breaker is Closed or Half-Open (never retry against an Open breaker), and only on the parts of the payment call that are safely idempotent (e.g., protected by an idempotency key), to avoid double-charging a customer.

**10. What happens if you set the Circuit Breaker's open-state timeout too short? Too long?**
Too short, and the breaker moves to Half-Open and re-admits traffic before the dependency has actually recovered, causing it to flip back to Open repeatedly ("flapping") and never giving the dependency real breathing room. Too long, and the breaker keeps failing fast on a dependency that has already recovered, needlessly degrading functionality and delaying restoration of full service.

**11. How do Circuit Breaker, Retry, and Bulkhead compose together in a single resilient call path?**
Bulkhead is applied first to bound concurrency and isolate resources for that dependency; the circuit breaker wraps the call to detect sustained failure and fail fast once tripped; retry logic wraps the outermost layer, using backoff and jitter to handle transient blips — but it should only fire while the circuit is Closed or Half-Open, since retrying against an Open breaker defeats the purpose of tripping it.

**12. Give two real-world systems/tools that implement these patterns, and note any important caveat.**
Resilience4j implements circuit breaker, retry, and bulkhead as composable decorators for the JVM, and is the widely recommended successor to Netflix's Hystrix, which is now in maintenance mode. Istio implements circuit breaking declaratively at the service mesh layer via outlier detection and connection pool limits, enforced by sidecar proxies without requiring application code changes; AWS SDKs also implement exponential backoff with jitter by default for retryable errors like throttling.
