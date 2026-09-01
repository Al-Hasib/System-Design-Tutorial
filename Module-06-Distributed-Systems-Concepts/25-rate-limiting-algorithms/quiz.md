# Practice & Interview Questions

**1. What problem does rate limiting solve, and why isn't "just add more servers" a sufficient answer?**
Rate limiting protects fairness (one client can't starve others), availability (guards against traffic spikes, retry storms, and abuse), and cost predictability (bounds how much capacity you must provision per client). Adding servers doesn't help because a single misbehaving or malicious client can scale its request volume faster than you can scale infrastructure, and unbounded traffic to a shared resource like a database can't be fixed by scaling the stateless layer alone.

**2. Explain the boundary problem in fixed window counters. How bad can it get?**
Fixed window resets its counter at fixed interval boundaries (e.g., every 60 seconds). A client can send the full limit right at the end of one window and the full limit again right at the start of the next, resulting in up to roughly 2x the intended limit within a short span straddling the boundary — even though each individual window's count stayed within limits.

**3. How does the sliding window log algorithm fix the boundary problem, and what does it cost you?**
It stores a timestamp for every request per client (e.g., in a Redis sorted set), and on each new request it discards entries older than the window and counts what remains. This gives exact enforcement with no boundary artifacts, but memory cost scales linearly with request volume per client, which becomes expensive at high throughput.

**4. What is the sliding window counter, and why is it the practical default in many systems?**
It keeps two fixed-window counters (current and previous) and estimates the sliding count as `current_count + previous_count * overlap_fraction`, where overlap_fraction is how much of the previous window still falls inside the sliding view. It approximates the precision of the sliding log while only costing two counters per client — combining low memory cost with good-enough accuracy.

**5. Why does token bucket allow bursts but leaky bucket doesn't?**
Token bucket controls the *rate of admission*: tokens accumulate up to a capacity while the client is idle, and a request only needs one token to be admitted, so a client can spend all banked tokens instantly in a burst. Leaky bucket controls the *rate of output*: requests queue and are drained to the backend at a strictly constant rate no matter how they arrived, so even a burst of arrivals is serialized out at the fixed leak rate — it can never exceed that rate.

**6. Design a distributed rate limiter that works consistently across 50 API gateway instances. What approach do you take and why?**
Use a centralized, atomically-updated store (typically Redis) as the single source of truth instead of local per-instance counters. Implement the check-and-increment as a single atomic operation — either Redis `INCR` + `EXPIRE` for fixed-window limiting, or a Lua script executed via `EVAL` for token bucket/sliding window logic — so the read-check-write sequence can't race across concurrent requests hitting different gateway instances. This guarantees the global limit is enforced exactly regardless of which instance a request lands on.

**7. What's the trade-off of putting Redis in the hot path of every rate-limited request?**
It adds a network round trip per request and makes rate limiting depend on Redis's availability and latency — if Redis is slow or down, every gated request is affected. The alternative is approximate local rate limiting (each instance enforces a fraction of the limit, or syncs counts periodically), which removes Redis from the hot path and improves latency/availability at the cost of allowing brief, bounded over-admission above the true global limit.

**8. A client complains their well-behaved app suddenly gets rate-limited even though it sends fewer than 100 requests/minute on average. What algorithm might be at fault, and why?**
Likely a fixed window counter combined with a client that happens to send its requests in a bursty pattern near the same wall-clock boundary each minute. Even though the average is low, the exact timing can straddle a window edge and trip the limit. Switching to a sliding window counter or token bucket would smooth this out since neither penalizes traffic based on a rigid clock-aligned boundary alone.

**9. How would you implement token bucket logic atomically in Redis? Sketch the approach.**
Store the bucket's last-refill timestamp and current token count as fields in a Redis hash per client key. In a Lua script (run via `EVAL` so it executes atomically): compute elapsed time since the last refill, add `elapsed * refill_rate` tokens capped at the bucket's max capacity, then if at least one token is available, decrement it and allow the request; otherwise reject. Because Redis executes Lua scripts single-threadedly, this avoids race conditions between concurrent requests from different gateway instances.

**10. Why do production systems like AWS API Gateway expose both a "rate" and a "burst" configuration value instead of a single limit?**
This maps directly to token bucket parameters: "rate" is the steady-state token refill rate (sustained throughput allowed indefinitely) and "burst" is the bucket capacity (how many requests can be admitted instantly from accumulated tokens). Exposing both lets API consumers handle realistic traffic patterns — occasional bursts (e.g., a client that batches work) — without raising the sustained rate limit and risking overload of backend capacity.

**11. When would you deliberately choose leaky bucket over token bucket for a production system?**
When the downstream system genuinely cannot tolerate bursts and needs a smoothed, constant-rate stream of work — for example, feeding a fixed-capacity worker queue, a legacy system with no buffering of its own, or shaping outbound traffic to a third-party API with a strict fixed-rate contract. Nginx's `limit_req` module is a real example: it queues and delays requests to enforce a constant outflow rate rather than letting bursts straight through.

**12. What happens to request semantics if you use a queue-based leaky bucket implementation versus a simple reject-on-full one?**
A queue-based (with `burst`) leaky bucket lets excess requests wait in a FIFO queue and get processed later at the constant leak rate, trading added latency for a lower rejection rate — good when clients can tolerate delayed responses. A simple reject-on-full leaky bucket immediately returns an error (e.g., `429`) once the bucket is full, prioritizing fast failure and predictable resource usage over completeness, which suits latency-sensitive systems where a stale, delayed response is worse than an immediate rejection.
