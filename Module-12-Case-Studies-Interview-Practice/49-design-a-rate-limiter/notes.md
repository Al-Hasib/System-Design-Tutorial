# Design a Rate Limiter — Interview Cheat Sheet

Quick-reference companion to [`README.md`](./README.md). Use this to rehearse the interview flow without re-reading the full script.

## 1. Requirements

**Functional**
- Per-user limits (authenticated traffic).
- Per-IP limits (anonymous/unauthenticated traffic).
- Per-API-key limits with configurable tiers (e.g., free = 100 req/min, paid = 10,000 req/min).
- Per-endpoint overrides (e.g., `/login` and `/password-reset` much stricter than `/search`).
- Standard `429 Too Many Requests` response with a `Retry-After` header.
- Rules configurable at runtime, no redeploy required.

**Non-functional**
- Must enforce correctly across the entire fleet, not per-instance ("distributed enforcement").
- Must add minimal latency — it runs on the hot path of every request.
- Must not become a single point of failure that turns a limiter outage into a full API outage.
- Must be fair/accurate, but availability and latency matter more than perfect precision.

## 2. Capacity Estimation

| Quantity | Estimate | Note |
|---|---|---|
| Peak global request rate | 10M req/sec | Across the whole API gateway fleet |
| Gateway instances | ~500 | ~20K req/sec per instance |
| Naive Redis ops needed for exact per-request check | 10M ops/sec | Single Redis node tops out ~100K-200K ops/sec → would need 50-100 shards |
| Unique rate-limit keys | 50M | Users + API keys + IP buckets |
| Memory per key (sliding window counter) | ~120 bytes | ~24B data + ~60-90B Redis per-key overhead |
| Total counter working set | ~6 GB | Easily shardable; memory is not the hard constraint — write throughput is |

**Takeaway:** exact, synchronous, check-every-request-against-one-store designs don't scale at this volume — this is the numeric justification for local pre-aggregation / approximate counting.

## 3. High-Level Architecture

```
Client → Load Balancer → API Gateway (rate-limiter middleware/sidecar)
                              │
                              ▼
                     Sharded Redis Cluster (counters, Lua scripts)
                              │
                     under limit │ over limit → 429 + Retry-After (short-circuit, never reaches downstream)
                              ▼
                     Downstream Microservices
```

- Rejects happen at the edge (gateway), not deep in the call graph.
- Optional client SDK does local optimistic throttling before a request is even sent.
- Optional service-mesh sidecars enforce service-to-service limits internally.

## 4. Key Design Decisions & Trade-offs

### Algorithm comparison

| Algorithm | Burst behavior | Output smoothness | Memory cost | Best used for |
|---|---|---|---|---|
| **Fixed Window** | Allows up to 2x limit at window boundary | Choppy, resets on a cliff | O(1) per key | Simple, coarse limits where boundary bursts are acceptable |
| **Sliding Window Log** | Precise, no boundary artifact | Exact | O(n) — one entry per request | Low-volume, high-precision limits |
| **Sliding Window Counter** | Close approximation of log | Smooth-ish (weighted estimate) | O(1) per key | Practical default — precision of log, cost of fixed window |
| **Token Bucket** | Allows bursts up to bucket capacity, then throttles to refill rate | Bursty then steady | O(1) per key | Client SDKs, API gateways — tolerating legitimate burstiness (Stripe, AWS API Gateway model) |
| **Leaky Bucket** | No burst pass-through; queues and drains at constant rate | Strictly uniform | O(1) per key + queue | Protecting fragile downstream systems that need smoothed input (Nginx `limit_req`) |

### Other decisions

- **Atomicity:** check-then-increment must be one atomic Redis Lua `EVAL`, not two round trips — avoids race conditions across concurrent requests on different gateway instances.
- **Consistency model:** favor Redis's availability/low-latency replication over strict linearizability — ties to CAP/PACELC; a rate limiter needs speed and uptime more than perfect global exactness.
- **Idempotency:** retried requests carrying an idempotency key should not double-consume quota.
- **Multi-layer enforcement:** client SDK (optimistic, local) → API gateway (authoritative, global, Redis-backed) → service mesh sidecars (internal service-to-service protection).
- **Hot keys:** a single viral user/API key can saturate one Redis shard regardless of overall cluster sharding — mitigate with local pre-aggregation and periodic sync instead of a Redis round trip per request.
- **Clock skew:** compute elapsed time using Redis's own server clock inside the Lua script rather than trusting per-instance client clocks.
- **Fail-open vs fail-closed:** circuit-break calls to the Redis/limiter store (see Circuit Breaker pattern). Fail-open (allow traffic) is the safer default for most public read traffic; fail-closed (reject traffic) is safer for security-sensitive endpoints like login, where an attacker benefits from the limiter being down.
