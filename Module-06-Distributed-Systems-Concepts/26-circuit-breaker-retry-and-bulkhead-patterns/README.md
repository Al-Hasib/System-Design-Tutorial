# Circuit Breaker, Retry & Bulkhead Patterns

Difficulty: Advanced | Estimated length: 15-20 min | Prerequisites: [Rate Limiting Algorithms](../25-rate-limiting-algorithms/README.md), [Availability, Reliability and Fault Tolerance](../../Module-01-Foundations/05-availability-reliability-and-fault-tolerance/README.md)

## Learning Objectives

- Explain how cascading failures propagate through a distributed system and why naive fault handling makes them worse.
- Describe the Circuit Breaker pattern's three states (Closed, Open, Half-Open), the thresholds that drive transitions, and how it protects a struggling dependency.
- Design a correct Retry strategy using exponential backoff and jitter, and explain why retries are only safe on idempotent operations.
- Apply the Bulkhead pattern to isolate failure domains via dedicated thread pools, connection pools, or semaphores.
- Combine Circuit Breaker, Retry, and Bulkhead into a single resilient call path and recognize their real-world implementations.

## Script

### Hook / Intro

Imagine one slow database query taking down your entire platform — not because the database crashed, but because every single request thread in your web servers is stuck waiting on it, and now nobody can even serve the homepage. That's not a hypothetical. That's a cascading failure, and it's one of the most common ways distributed systems die in production. Today we're covering the three patterns that stop this from happening: Circuit Breaker, Retry, and Bulkhead. These are the seatbelts and airbags of distributed systems — you hope you never need them, but the day you do, they're the only thing standing between a minor blip and a full outage.

### Cascading Failures — The Problem

Here's how it usually starts. You have Service A calling Service B, which calls a downstream Service C. Service C starts responding slowly — maybe it's under load, maybe a disk is failing, maybe a network link is degraded. It's not fully down, which is actually the worst case, because a hard failure fails fast. A slow dependency instead ties up resources.

Service B's threads start blocking waiting on C. Its thread pool fills up. Now B can't serve any requests — even ones that don't touch C — because there are no threads left. Service A sees B timing out, and if A retries aggressively without any restraint, it just adds more load onto an already struggling B and C. This is the thundering herd problem compounding a failure. Within seconds, a slowdown in one low-level dependency turns into total unavailability several layers up the call stack. This is a cascading failure, and it's exactly what these three patterns exist to prevent.

### The Circuit Breaker Pattern

The Circuit Breaker pattern is directly borrowed from electrical engineering — a physical breaker trips to stop current flow when there's a fault, protecting the rest of the circuit. In software, it wraps a call to a remote dependency and tracks its success and failure rate.

There are three states. The first is **Closed** — the normal state. Requests flow through to the dependency as usual, and the breaker just counts failures, typically over a rolling window, like "the last 20 requests" or "the last 10 seconds." If the failure rate crosses a configured threshold — say, more than 50% of requests failing, or five consecutive timeouts — the breaker trips into the **Open** state.

In the Open state, the breaker stops calling the dependency entirely. Every call fails immediately — fast, cheap, in-process — instead of waiting on a timeout. This is the crucial insight: failing fast frees up threads and connections instead of letting them pile up. The breaker stays Open for a configured timeout period, often called the "sleep window," maybe 30 seconds or a minute.

After that timeout expires, the breaker moves to **Half-Open**. This is a probing state — it allows a small number of test requests through to see if the dependency has recovered. If those test calls succeed — meeting some success threshold, like 3 out of 3, or a certain success percentage — the breaker transitions back to Closed and normal traffic resumes. If they fail, it immediately flips back to Open and waits another full timeout period before probing again. This half-open probing is what prevents the breaker from just slamming a barely-recovering service with full traffic the instant its timeout expires.

The tuning knobs that matter here are: the failure-rate threshold, the size of the rolling window used to calculate that rate, the open-state timeout duration, and the number and success criteria of the half-open probe requests. Get these wrong — too sensitive, and you trip on minor blips; too lax, and you don't protect anything.

### Retry Pattern

Retries seem simple — a call failed, try it again — but naive retries are actually dangerous, especially during an outage. If a service is struggling and every client immediately retries on failure, you've just multiplied your load on a system that was already failing. This is why retry logic needs real structure.

The standard approach is **exponential backoff**: instead of retrying immediately, you wait progressively longer between attempts — say 100ms, then 200ms, then 400ms, then 800ms, doubling each time up to some maximum. This gives the downstream service breathing room to recover instead of getting hammered continuously.

But exponential backoff alone has a flaw: if thousands of clients all failed at the same moment — say, due to a brief network blip — they'll all back off on the exact same schedule and all retry at the exact same moment again. That's called the thundering herd problem, and it's solved with **jitter** — adding randomness to each backoff interval so retries get spread out over time instead of arriving in synchronized waves. AWS's well-known "full jitter" approach picks a random delay between zero and the computed exponential backoff value for each attempt.

You also want a **retry budget** — a cap on total retries, either as an absolute count or a percentage of overall request volume, so retries never become the majority of your traffic during a bad outage. And critically: retries are only safe on **idempotent** operations — an operation that produces the same result no matter how many times it's applied. Retrying a GET request is safe. Retrying a "charge this credit card" POST without an idempotency key can double-charge a customer. This is why real-world retry systems are almost always paired with idempotency keys on the server side, so a duplicate request is recognized and safely deduplicated rather than reapplied.

### Bulkhead Pattern

The Bulkhead pattern takes its name from ship design — a ship's hull is divided into watertight compartments, so if one compartment floods, the water is contained there and doesn't sink the whole vessel. Apply that same idea to software: isolate resources per-dependency so that one failing dependency can't exhaust resources needed by everything else.

Concretely, this means giving each downstream dependency its own dedicated thread pool, connection pool, or semaphore, rather than sharing one common pool across all calls. Remember our cascading failure scenario earlier? If Service B had used a separate, bounded thread pool just for calls to the slow Service C — say, capped at 10 threads — then even if all 10 of those threads got stuck waiting on C, the rest of B's thread pool remains free to serve every other type of request. The blast radius of C's slowness is contained to just the C-calling code path.

Bulkheads can be implemented at multiple levels: thread pool isolation (each dependency gets its own executor), semaphore isolation (a lightweight counter limiting concurrent calls without a full pool, useful for very high volume calls), connection pool isolation (a database or HTTP client gets a capped, dedicated connection pool per downstream target), and even at the infrastructure level — separate service instances or node pools for different workload classes, so a noisy or misbehaving tenant can't consume all the shared capacity.

### Combining the Patterns

These three patterns are complementary, not competing — production systems use them together. A typical resilient call wraps a downstream call first in a **bulkhead** to cap concurrency and isolate resources, then in a **circuit breaker** to detect sustained failure and fail fast, and the retry logic sits around the whole thing, but only retrying when the breaker is Closed or Half-Open — never retrying against an Open breaker, because that defeats the purpose of tripping it in the first place. The ordering matters: bulkhead limits concurrency, circuit breaker prevents wasted calls to a known-bad dependency, and retry with backoff and jitter handles the transient, recoverable blips that remain.

### Real-World Example

You don't have to build these from scratch. Netflix pioneered a lot of this thinking with **Hystrix**, their circuit breaker library that popularized this pattern in the microservices world — Hystrix is now in maintenance mode, and its widely recommended successor is **resilience4j**, a lightweight Java library offering circuit breakers, retry, bulkheads, and rate limiters as composable decorators. Outside the JVM world, the **AWS SDKs** implement exponential backoff with jitter by default for retryable errors like throttling responses, so you get sane retry behavior out of the box when calling AWS services. And at the infrastructure layer, service meshes like **Istio** implement circuit breaking declaratively — you configure outlier detection and connection pool limits on a Kubernetes service, and the mesh's sidecar proxies enforce ejection of unhealthy endpoints without your application code needing to know anything about it at all.

### Recap

Let's tie it together. Cascading failures happen when a slow or failing dependency exhausts shared resources up the call chain. The Circuit Breaker pattern stops that by failing fast once a failure threshold trips, entering an Open state, then cautiously probing recovery through a Half-Open state before fully closing again. The Retry pattern handles transient failures safely using exponential backoff with jitter and a retry budget, but only on idempotent operations. And the Bulkhead pattern isolates resource pools per dependency, like watertight compartments in a ship, so one dependency's failure can't sink everything else. Together, they're the core toolkit for building systems that degrade gracefully instead of collapsing entirely.

### What's Next

Next up, we're moving into the deep end of distributed systems theory: Consensus Algorithms — Paxos and Raft. We'll unpack how distributed nodes agree on a single source of truth even when some of them fail, which is the foundation underneath everything from distributed databases to leader election in Kubernetes. See you there.

## Key Takeaways

- Cascading failures spread because a slow dependency ties up shared resources (threads, connections) all the way up the call chain.
- A Circuit Breaker has three states — Closed (normal), Open (fail fast, no calls sent), and Half-Open (limited probe requests) — driven by a failure-rate threshold, an open-state timeout, and a half-open success threshold.
- Retries must use exponential backoff with jitter to avoid synchronized thundering-herd retries, and should be bounded by a retry budget.
- Retries are only safe for idempotent operations; non-idempotent operations need idempotency keys to be retried safely.
- Bulkheads isolate resource pools (thread pools, connection pools, semaphores) per dependency so one failing dependency can't starve the rest of the system, just like watertight compartments in a ship.
- In production, bulkhead, circuit breaker, and retry are combined — retries should never fire against an Open circuit.
- Resilience4j (successor to Netflix Hystrix), AWS SDK retry behavior, and Istio's circuit breaking are real, widely used implementations of these patterns.
