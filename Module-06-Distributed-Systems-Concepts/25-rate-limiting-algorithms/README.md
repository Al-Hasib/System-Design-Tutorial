# Rate Limiting Algorithms (Token Bucket, Leaky Bucket, Sliding Window)

**Difficulty:** Advanced | **Estimated length:** 15-20 min | **Prerequisites:** [API Gateway and BFF Pattern](../../Module-02-Networking-and-Communication/09-api-gateway-and-bff-pattern/README.md), [Distributed Caching: Redis and Memcached](../../Module-04-Caching-and-Content-Delivery/19-distributed-caching-redis-and-memcached/README.md)

## Learning Objectives

- Explain why rate limiting is necessary for protecting APIs and backend resources.
- Compare Fixed Window, Sliding Window Log, Sliding Window Counter, Token Bucket, and Leaky Bucket algorithms.
- Reason precisely about burst tolerance and traffic-smoothing trade-offs between algorithms.
- Design a distributed rate limiter that stays consistent across many API gateway or service instances using Redis.
- Recognize how real systems (Stripe, AWS API Gateway, Nginx) implement rate limiting in production.

## Script

### Hook / Intro

Picture this: it's Black Friday, your API is getting hammered, and one misbehaving client script is firing ten thousand requests a second at your `/checkout` endpoint. No rate limiting in place. Your database connection pool saturates, latency spikes for every legitimate customer, and now everyone's having a bad day because of one bad actor. This is the problem rate limiting solves. Today we're going deep on the algorithms that make it work — token bucket, leaky bucket, and the sliding window family — and we're going to get precise about the thing most tutorials wave their hands at: how you enforce rate limits consistently when you've got fifty gateway instances behind a load balancer, not just one.

### Why Rate Limiting Matters

Rate limiting caps how many requests a client — identified by API key, user ID, or IP — can make in a given time window. You've already seen API gateways in this series; rate limiting is one of the core responsibilities that lives there, alongside auth and routing. Why do you need it? Three big reasons. First, fairness — one noisy tenant shouldn't starve every other tenant sharing the same infrastructure. Second, protection — it's your first line of defense against traffic spikes, retry storms, and denial-of-service style abuse, deliberate or accidental. Third, cost and capacity planning — if you know each client is capped at, say, 100 requests per minute, you can actually size your downstream services and databases with confidence. The algorithm you pick determines two things: whether you allow bursts of traffic, and how smooth or spiky the traffic reaching your backend actually looks.

### Fixed Window Counter

The simplest approach: pick a window size, say 60 seconds, and keep a counter per client that increments on every request. When the counter exceeds your limit — 100 requests, say — you reject further requests until the window resets. Dead simple, O(1) memory per client, trivial to implement with a single Redis key and a TTL. But it has a well-known flaw: the boundary problem. If a client sends 100 requests in the last second of one window and another 100 in the first second of the next window, that's 200 requests in a two-second span, even though the configured limit was 100 per minute. Fixed window doesn't smooth anything — it just resets a cliff every interval, and traffic can burst hard right at that boundary.

### Sliding Window Log and Sliding Window Counter

Sliding window log fixes the boundary problem properly: for each client, store a timestamp for every request in a sorted set, and when a new request comes in, drop entries older than your window and count what's left. It's precise — no boundary artifacts — but the memory cost scales with request volume, since you're storing a timestamp per request. That's expensive at high throughput. Sliding window counter is the practical compromise: instead of logging every timestamp, you keep counters for the current and previous fixed window, then compute a weighted count — something like "current window count plus previous window count times the fraction of the previous window still overlapping the sliding view." It approximates the precision of the log approach with the memory footprint of fixed window counters. This is what most production systems actually use when they want sliding-window semantics without the storage blowup.

### Token Bucket

Now the algorithm most engineers reach for by default: token bucket. Imagine a bucket that holds tokens, with a capacity — say 100 tokens. Tokens refill at a steady rate, say 10 per second, up to that capacity. Every incoming request has to grab one token to proceed; if the bucket's empty, the request is rejected or queued. Here's the key behavior: if the client has been idle and the bucket is full, it can suddenly fire off 100 requests in a burst, instantly, and they'll all succeed, because the tokens were sitting there accumulated. Then it's throttled back to the steady refill rate of 10 per second. Token bucket explicitly allows bursts up to the bucket capacity — that's a feature, not a bug, for a lot of real-world traffic patterns, like a client that polls occasionally then does a batch of work.

### Leaky Bucket

Leaky bucket flips the mental model. Think of an actual bucket with a small hole in the bottom, leaking water at a constant rate. Requests pour in at the top, at whatever rate the client sends them. If the bucket fills up — meaning requests arrive faster than the leak rate — new requests overflow and get dropped or rejected. But critically, water leaks out, meaning requests get processed, at a fixed, constant rate, no matter how bursty the input was. This is usually implemented as a FIFO queue with a fixed-rate worker draining it. The consequence: leaky bucket smooths traffic into a strictly uniform outflow rate. It does not allow bursts to pass through faster than the leak rate — even if the bucket has capacity, a sudden burst just queues up and gets drained slowly. That's the core distinction from token bucket: token bucket allows bursts to be forwarded immediately as long as tokens are banked; leaky bucket always outputs at a constant rate regardless of how the input arrived. Pick token bucket when you want to tolerate burstiness from legitimate clients; pick leaky bucket when your downstream system genuinely needs a smoothed, constant-rate stream — think traffic shaping into a fixed-capacity queue or a legacy system that can't handle spikes at all.

### Distributed Rate Limiting (multi-server, using Redis)

Here's the part that actually separates a junior answer from a senior one. All of this is easy when there's one process holding the counter in memory. But your API gateway runs as fifty instances behind a load balancer, and a given client's requests can land on any of them. If each instance keeps its own local token bucket, a client could get 100 requests through gateway instance A, then another 100 through instance B, and you've silently allowed 5000% of your intended limit. You need shared, synchronized state. The standard solution is a centralized counter in Redis. The trick is atomicity: incrementing a counter and checking it against a limit is a read-then-write, and if you do that as two separate round trips you get race conditions under concurrency. So you use Redis's atomic `INCR` with a TTL for fixed-window style limiting, or, more robustly, a Lua script that implements token bucket or sliding window counter logic entirely inside Redis, executed atomically in a single round trip via `EVAL`. Redis's single-threaded execution model guarantees no interleaving. For token bucket specifically, the Lua script computes elapsed time since the last refill, adds the appropriate number of tokens, caps at capacity, and atomically decrements if a token is available — all server-side. This gives you exact, globally consistent enforcement, but it adds a network hop and a dependency on Redis being available and low-latency for every single request. The alternative, when you're willing to trade precision for lower latency and higher availability, is approximate local rate limiting: each gateway instance enforces limit-divided-by-instance-count locally, or periodically syncs counts to a shared store rather than checking on every request. This is eventually consistent — a client might briefly exceed the global limit — but it removes Redis from the hot path of every request. The choice is a classic consistency-versus-latency trade-off, and which one you pick depends on whether slightly over-admitting requests for a few hundred milliseconds is acceptable for your use case.

### Real-World Example

You don't have to imagine this — production systems make these trade-offs explicitly. Stripe's API documents a rate limit per account and returns a `429 Too Many Requests` with `Retry-After` guidance, effectively a token-bucket-style model that tolerates short bursts while enforcing a steady long-run rate. AWS API Gateway lets you configure both a steady-state rate and a burst capacity per API, which is literally token bucket terminology exposed directly in the console — rate is the refill rate, burst is the bucket size. Nginx implements rate limiting via the `ngx_http_limit_req_module`, which is a leaky bucket implementation by design — it queues and delays requests to smooth them to a configured rate, with an optional `burst` parameter that lets a controlled number of requests queue up rather than being rejected outright. Notice how all three real systems blend token-bucket-like burst allowances with leaky-bucket-like smoothing — the pure textbook algorithms are a starting point, but production configs usually combine ideas from both.

### Recap

Let's tie it together. Fixed window is simple but has a boundary burst problem. Sliding window log is precise but memory-hungry. Sliding window counter approximates the log with fixed-window-like memory cost — the practical default for many teams. Token bucket allows controlled bursts up to bucket capacity, then throttles to a steady refill rate — great for tolerating legitimate burstiness. Leaky bucket forces a constant output rate regardless of input burstiness — great when downstream truly needs smooth, uniform traffic. And the hard distributed-systems part: making any of these work correctly across many stateless gateway instances requires either a centralized, atomically-updated store like Redis with Lua scripts, or an accepted approximation via local counters for lower latency.

### What's Next

Rate limiting protects you from too many requests. But what happens when a downstream service is slow or failing outright, and your own service keeps calling it anyway, making things worse? That's the next video: Circuit Breaker, Retry, and Bulkhead patterns — how to build resilience into service-to-service calls so one failing dependency doesn't cascade into a full outage. See you there.

## Key Takeaways

- Rate limiting protects fairness, availability, and cost predictability by capping request rates per client.
- Fixed window is simplest but allows boundary bursts of up to 2x the intended limit.
- Sliding window log is exact but O(n) in memory per client; sliding window counter approximates it cheaply.
- Token bucket allows bursts up to bucket capacity, then throttles to the refill rate — good for bursty legitimate traffic.
- Leaky bucket enforces a constant output rate regardless of input burstiness — good for smoothing traffic into fragile downstream systems.
- Distributed rate limiting requires shared, atomically-updated state (e.g., Redis `INCR` or Lua scripts) to avoid per-instance over-admission; local approximations trade consistency for latency and availability.
- Real systems (Stripe, AWS API Gateway, Nginx) blend burst allowance and smoothing rather than using one pure textbook algorithm.
