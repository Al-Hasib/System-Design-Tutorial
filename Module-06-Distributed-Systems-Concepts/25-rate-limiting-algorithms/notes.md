# Study Notes: Rate Limiting Algorithms

## Comparison Table

| Algorithm | Allows Bursts? | Smooths Traffic? | Memory Cost | Implementation Complexity | Common Use Case |
|---|---|---|---|---|---|
| Fixed Window Counter | Yes, at window boundaries (up to ~2x limit) | No | Low — 1 counter per client | Low | Simple per-key limits (e.g., 100 req/min per API key) |
| Sliding Window Log | No (precise enforcement) | Somewhat (exact window) | High — O(n) timestamps per client | Medium-High | Precision-critical limits at moderate volume |
| Sliding Window Counter | Small, bounded burst at boundary | Yes (approximate) | Low — 2 counters per client | Medium | Practical default for API gateways (approximates log cheaply) |
| Token Bucket | Yes, up to bucket capacity | No (bursts pass through immediately) | Low — tokens + timestamp per client | Medium | APIs that tolerate bursty legitimate clients (Stripe, AWS API Gateway) |
| Leaky Bucket | No (constant output rate) | Yes (strictly constant) | Low-Medium — queue + timestamp per client | Medium | Traffic shaping into fragile/fixed-capacity downstream systems (Nginx) |

## Key Numbers / Examples

- Typical API rate limit: "100 requests/minute per API key" or "1000 requests/hour per user."
- Stripe: limits vary by endpoint/mode, enforced per account, returns `429` with `Retry-After` header.
- AWS API Gateway: configurable steady-state rate (req/sec) + burst capacity (token bucket terms exposed directly).
- Nginx `limit_req_zone`: rate like `10r/s` with an optional `burst=20` parameter to queue excess requests instead of rejecting immediately.
- Redis `INCR` + `EXPIRE` pattern: O(1) atomic fixed-window counter; a single round trip via Lua script (`EVAL`) is needed for token bucket / sliding window counter to keep the read-check-write atomic.

## Interview Revision — Bullet Summary

- Fixed window: cheapest, but has a boundary problem — traffic can burst to ~2x the limit across a window edge.
- Sliding window log: exact, stores every request timestamp, memory scales with traffic — rarely used at scale as-is.
- Sliding window counter: weighted average of current + previous fixed windows — practical approximation of the log, fixed-window-like memory.
- Token bucket: tokens refill at a steady rate up to a capacity; a request consumes one token; allows a burst up to bucket size when idle tokens have accumulated.
- Leaky bucket: requests queue and drain at a strictly constant rate; excess requests overflow/reject; never bursts past the leak rate regardless of input pattern.
- Token bucket vs leaky bucket, one-line distinction: token bucket controls the *rate of admission* and permits saved-up capacity to be spent instantly (bursty output); leaky bucket controls the *rate of output* and always smooths to a constant rate (non-bursty output).
- Distributed rate limiting needs shared state across instances — use Redis with atomic `INCR`/Lua scripts for exact global enforcement, or accept eventual consistency with local per-instance counters for lower latency and no single point of failure/bottleneck.
- Rate limiting typically lives in the API gateway layer (see API Gateway/BFF video) and often relies on a distributed cache like Redis (see Distributed Caching video) as the shared state store.
