# Why This Topic Matters: Design a Rate Limiter

> **In one sentence:** The single-server version is twenty lines of code, which is exactly why this problem is asked — everything interesting only appears once you have more than one server, and that is where the interview actually starts.

## Why This Case Study Exists

Most design prompts are about building a product. This one is about building a *component* — one that sits on the hot path of every request, must add almost no latency, must be correct under concurrency, and must keep working when its own dependencies fail.

It is also unusually honest about the nature of distributed systems. The naive solution works perfectly on one machine. Scale to ten machines and it is silently wrong by a factor of ten, with no error and no alert. That gap between "obviously correct" and "quietly broken at scale" is the lesson.

## The Design Problems It Forces You to Solve

### 1. Counting correctly across many servers
**The problem:** Ten API servers, each tracking "requests per user per minute" in local memory. Your 100/minute limit is actually 1000/minute.

**Why it is hard:** Correctness requires shared state, and shared state on the hot path of every request costs latency and creates a dependency.

**What you learn:** The options and their real trade-offs. **Centralized counters in Redis** are accurate and add a network round trip plus a critical dependency. **Local counters at 1/N of the limit** are fast and wrong when load is uneven across servers. **Local counters with periodic synchronization** are a middle ground that is approximately right and eventually consistent. Naming the trade-off rather than assuming Redis is the whole point — and if you do choose Redis, knowing that the increment-and-check must be atomic (a Lua script or a pipelined `INCR` with `EXPIRE`) rather than a read-then-write race.

### 2. Choosing an algorithm for the actual traffic shape
**The problem:** A fixed-window counter of 100/minute permits 200 requests in one second across a window boundary.

**Why it is hard:** Every algorithm has a distinct failure mode, and the right one depends on what you are protecting and how bursty legitimate traffic is.

**What you learn:** **Token bucket** allows bounded bursts while capping the long-run rate — usually the best default, because real client traffic is bursty and throttling a legitimate burst generates support tickets. **Leaky bucket** enforces a strictly smooth output rate, which is what you want when the thing downstream has fixed capacity. **Sliding window log** is exact and stores a timestamp per request, which is too much memory at scale. **Sliding window counter** approximates the log cheaply and fixes the boundary problem. Being able to pick one and justify it against the traffic pattern is what is being graded.

### 3. Deciding what to do when the limiter itself fails
**The problem:** Redis is unavailable. Do you allow all traffic or reject all traffic?

**Why it is hard:** Both answers are bad. Failing open means an unprotected system during an incident, exactly when it is most fragile. Failing closed means your rate limiter causes a total outage.

**What you learn:** That this is a deliberate product decision, not a technical one, and that it usually differs per endpoint — fail open for ordinary API traffic (availability matters more than perfect limits), fail closed for login and payment endpoints (where the limit *is* the security control). Having a position here, and knowing it varies, is a strong signal.

### 4. Identifying the client, and telling them what happened
**The problem:** Limiting by IP punishes everyone behind a corporate NAT or a mobile carrier gateway and is trivially bypassed with a proxy pool. Limiting by API key requires authentication first, which unauthenticated endpoints do not have.

**What you learn:** Layered identity — API key where available, user ID where authenticated, IP as a coarse fallback with a higher threshold — and different limits per tier. Plus the client-facing contract: return `429 Too Many Requests` with `Retry-After`, and ideally `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` so well-behaved clients can self-throttle instead of hammering you into a retry storm.

## What It Costs to Get Wrong

- **Answering only the single-server version** and not raising the distributed problem is the most common way this interview goes poorly.
- **Read-then-write on the counter** is a race that under-counts exactly when traffic is highest.
- **Choosing an algorithm without justification** reads as memorization.
- **Ignoring the failure mode** of the limiter leaves a component on every request path with undefined behavior during an incident.
- **Over-restricting** produces a limiter that is technically correct and commercially harmful, because legitimate bursts are part of normal usage.

## Why Interviewers Choose This One

It is compact enough to finish and deep enough to probe. It tests algorithm selection under constraints, distributed state management, atomic operations, failure-mode reasoning, and API design — five things, in one question, with no domain knowledge required. It is also a component that exists in nearly every real system, so the discussion stays concrete rather than hypothetical.

## How It Connects

This is the applied version of **rate limiting algorithms** (topic 25), implemented on **Redis** with atomic operations (topic 19), deployed at the **API gateway** or **reverse proxy** (topics 9, 8). It is a core **security** control against brute force (topic 36), complements **circuit breakers** (topic 26) on the other side of the call, and at very high key cardinality it reaches for **Count-Min Sketch** (topic 42). The fail-open/fail-closed question is a direct instance of the **availability versus correctness** trade-off from **CAP** (topic 15).

**Next:** [Design a Chat Application](../50-design-a-chat-application-whatsapp/why.md) — where connection state and delivery guarantees become the whole problem.
