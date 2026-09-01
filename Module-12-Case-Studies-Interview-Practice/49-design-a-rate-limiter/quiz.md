# Follow-Up Interview Questions — Design a Rate Limiter

Use these to rehearse beyond the main script in [`README.md`](./README.md). Each answer is intentionally concise — expand out loud in a real interview.

---

**1. How would you rate-limit across multiple data centers, not just multiple gateway instances in one region?**

Don't try to keep one global, strongly-consistent counter across regions — the cross-region round trip would blow your latency budget. Instead, give each data center a local Redis cluster with a per-region share of the global limit (e.g., global limit ÷ number of regions, weighted by expected regional traffic), and optionally reconcile asynchronously in the background. This trades perfect global precision for regional low latency — the same availability-over-consistency call as the single-region design, just applied one level up.

---

**2. What happens when Redis goes down? Walk through the failure end to end.**

Calls from the gateway to Redis are wrapped in a circuit breaker. After enough consecutive failures/timeouts, the breaker trips and stops calling Redis, failing fast instead of piling up latency. Then policy takes over: fail-open (allow requests through, using a conservative local in-memory fallback limit) for most public/read traffic where availability matters more; fail-closed (reject requests) for sensitive endpoints like login, where letting unlimited traffic through during an outage is dangerous. The breaker periodically probes Redis (half-open state) to detect recovery and resume normal enforcement.

---

**3. Sliding window vs fixed window — what specifically goes wrong at the boundary with fixed window?**

Fixed window resets the counter to zero at a fixed interval boundary. A client can send the full limit right before the boundary and the full limit again right after, producing up to 2x the intended limit within a short span straddling the reset — because the two bursts land in two different "windows" that don't overlap in the counter's view, even though they overlap in real time.

---

**4. How does the sliding window counter approximate the sliding window log without storing every timestamp?**

It keeps two fixed-window counters — current and previous — and computes a weighted estimate: current window count plus previous window count multiplied by the fraction of the previous window still inside the sliding view. It's an approximation, not exact, but it gets very close to true sliding-window behavior while staying O(1) in memory per key instead of O(n) per request.

---

**5. Should you enforce a global rate limit, a per-endpoint limit, or both?**

Both, typically. A global per-client limit protects overall fairness and cost, while per-endpoint overrides protect specific hot or sensitive paths (e.g., `/login`, `/search`, a heavy report-generation endpoint) that need stricter or looser limits than the account's baseline. Implement per-endpoint as an override layered on top of the global check, not a replacement for it.

---

**6. How do you handle a legitimate burst of traffic without either rejecting good users or letting abuse through?**

Use token bucket rather than fixed/sliding window when burst tolerance is a requirement — it explicitly allows a client to spend up to the bucket's full capacity in one burst if it's been idle and accumulated tokens, then throttles to the steady refill rate. Pair this with a slightly higher short-term burst allowance than the sustained rate (the same rate + burst split AWS API Gateway exposes) so a legitimate momentary spike isn't punished the same as sustained abuse.

---

**7. How do you distinguish malicious traffic from a legitimate spike (e.g., a flash sale)?**

Pure request-rate limiting can't fully distinguish intent — that requires signals beyond rate: request patterns (uniform bot-like timing vs human variability), source diversity (one IP/API key vs many distinct legitimate users), and behavioral anomaly detection layered on top of rate limiting. In an interview, it's reasonable to say the rate limiter's job is to cap damage regardless of intent, while a separate abuse-detection/WAF layer handles the classification problem — and that expected legitimate spikes (a flash sale) should get a pre-provisioned higher limit tier rather than relying on the limiter to "guess."

---

**8. Why use an atomic Lua script in Redis instead of separate GET/INCR calls from the application?**

A separate read-then-write is two network round trips with a race window in between — under concurrency, two requests can both read a count just under the limit and both get admitted, silently exceeding it. A Lua script executed via `EVAL` runs entirely inside Redis's single-threaded execution engine as one atomic unit, so the read, refill/window math, and decrement can never interleave with another client's script.

---

**9. How do you avoid double-charging a client's quota when it retries a timed-out request?**

Tag retryable requests with an idempotency key. The rate limiter (or the service behind it) recognizes a repeated key within a bounded window as a duplicate of an already-counted request rather than a new one, so a network-level retry doesn't consume two units of quota for what is logically a single operation. This is the same idempotency mechanism used for safe retries in general, just applied to the quota-counting path specifically.

---

**10. What is the hot-key problem here, and how do you mitigate it?**

Even with a well-sharded Redis Cluster, all requests for one specific key (a viral account, a single heavily-used API key) hash to one shard, so that shard's load doesn't spread out no matter how many total shards exist. Mitigations: local pre-aggregation at each gateway instance with periodic sync to Redis instead of a Redis round trip per request, splitting one hot logical key into several sub-keys with a small in-process reconciliation, or giving known high-volume keys a dedicated, over-provisioned shard.

---

**11. How does clock skew across gateway instances affect a distributed rate limiter, and how do you avoid it?**

If each gateway instance computes elapsed time or window boundaries using its own local clock, skew between instances can cause inconsistent refill/reset timing — one instance thinks a window rolled over slightly before another does. The fix is to compute all time-dependent logic (elapsed time since last refill, current window index) using Redis's own server clock inside the atomic Lua script, so there is one single source of truth for time rather than trusting NTP-synced-but-still-imperfect client clocks.

---

**12. Where would you enforce rate limits: client SDK, API gateway, or service mesh — and why not just pick one?**

Each layer solves a different problem. A client SDK enforces a local, optimistic limit to give instant feedback and avoid wasting a network round trip on a request that's obviously going to be rejected — but it's advisory, since a malicious client can just skip the SDK. The API gateway enforces the authoritative, global limit against the shared Redis store — this is the one that actually matters for correctness. Service mesh sidecars enforce internal service-to-service limits so that one overloaded internal caller can't starve a shared internal dependency, which the gateway never sees since that traffic doesn't cross the edge. You need the gateway; the other two are defense in depth and UX.
