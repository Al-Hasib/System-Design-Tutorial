# Why This Topic Matters: Rate Limiting Algorithms

> **In one sentence:** Any endpoint open to the internet will eventually be hit far harder than you planned — by an attacker, a buggy client, or an enthusiastic customer — and without a limiter the only thing that stops it is your system falling over.

## The World Before This Idea

No rate limits. Consider what's now possible:

- A script tries 50,000 passwords per minute against your login endpoint.
- One customer's retry loop has no backoff, so a transient error turns into 10,000 requests per second, permanently.
- A scraper pulls your entire catalog, using more capacity than all real users combined.
- Someone signs up 100,000 fake accounts overnight.
- A single expensive API call (a report, a search, an export) is called in a loop and starves everything else.

None of these require malice — the buggy-client case is the most common by a wide margin — and all of them have the same outcome: one caller consumes capacity that belongs to everyone, and the system degrades or dies for all users.

## The Problems It Solves

### 1. One caller degrading service for everyone
**What you see:** Latency spikes and errors across the board; investigation shows a single API key responsible for 80% of traffic.

**Why it happens:** Capacity is shared and unallocated. Without a limit, first-come-first-served means whoever asks fastest gets everything.

**How rate limiting solves it:** Per-client quotas convert a shared resource into allocated shares. One client hitting its limit gets 429s; everyone else is unaffected. This fairness property — not raw protection — is the everyday value.

### 2. Brute force and enumeration
**What you see:** Credential stuffing against login, promo codes guessed exhaustively, user IDs enumerated to scrape private data.

**Why it happens:** These attacks depend entirely on being able to make an enormous number of attempts cheaply.

**How rate limiting solves it:** It makes each attempt cost time. Five login attempts per minute per account turns a 50,000-attempt attack from a two-minute job into an infeasible one. For security-sensitive endpoints, rate limiting *is* the control.

### 3. Cost amplification
**What you see:** A surprise five-figure bill from an SMS provider, an LLM API, or cloud egress.

**Why it happens:** Every call you forward costs real money, and an unbounded caller means an unbounded bill.

**How rate limiting solves it:** Limits become a spend ceiling. This applies inward (protecting yourself from your users) and outward (protecting yourself from your own retry storms hitting a paid third-party API).

### 4. Burst behavior that a naive limiter gets wrong
**What you see:** A fixed-window limiter of 100/minute lets a client send 100 requests at 11:59:59 and another 100 at 12:00:00 — 200 requests in one second, double the intended rate, right at the boundary.

**Why it happens:** Fixed windows reset abruptly, so the boundary is exploitable.

**How the algorithms solve it:** This is why the algorithm choice matters. **Token bucket** allows controlled bursts (tokens accumulate up to a cap) while bounding the long-run average — usually the best default because real traffic is bursty and users hate being throttled on a legitimate burst. **Leaky bucket** enforces a strictly smooth output rate, which is what you want when protecting a downstream system with a fixed capacity. **Sliding window log** is exact but memory-hungry; **sliding window counter** approximates it cheaply and fixes the boundary problem. Each is the right answer to a different question.

## The Price You Pay

- **Legitimate users get blocked.** Limits set too low break real workflows, and the resulting support load is a genuine cost. Tiered limits, burst allowances, and clear `Retry-After` headers are how you mitigate it.
- **Distributed counting is a real problem.** With ten API servers, each counting locally, your "100/minute" limit is actually 1000/minute. Making it correct requires shared state (Redis) on the hot path of every request — which adds latency and a critical dependency. The alternatives (local limits at 1/N, or approximate sync) trade accuracy for speed, and that trade should be conscious.
- **Identifying the client is harder than it looks.** IP-based limiting punishes everyone behind a corporate NAT or mobile carrier gateway, and is trivially evaded with a proxy pool. API keys are better but require authentication before limiting — which means unauthenticated endpoints need a different strategy.
- **The limiter itself can fail.** If Redis is down, do you fail open (no limits, risking overload) or fail closed (reject everything, causing an outage)? There's no universally right answer; there's only the answer you chose deliberately in advance.
- **It's not DDoS protection.** A volumetric attack saturates your network before your application-layer limiter ever runs. That needs upstream filtering.

## When You Need It — and When You Don't

| Rate limit when | It's less critical when |
|---|---|
| The endpoint is publicly reachable | It's internal-only behind a trusted network |
| Requests are expensive (compute, money, third-party calls) | The operation is trivially cheap and idempotent |
| It's security-sensitive (login, password reset, signup, OTP) | — |
| You offer tiered plans and need quota enforcement | — |
| You're protecting a fragile downstream dependency | — |

## Why This Shows Up in Interviews

Rate limiting is both a component in most designs (it belongs at the API gateway in essentially every diagram) and a full standalone design question — topic 49 in this course is exactly that. Interviewers probe: which algorithm and why, where does the counter state live, how does it work across many servers, what do you return to the client (429 plus `Retry-After`), and what happens when the rate-limiting store is unavailable. The distributed-counting problem is the crux, and candidates who jump straight to it demonstrate they've thought past the single-server version.

## How It Connects

Rate limiting lives at the **API gateway** (topic 9) or **reverse proxy** (topic 8), is almost always implemented on **Redis** (topic 19) for shared atomic counters, and is a core **security** control (topic 36). It complements **circuit breakers** (topic 26) — a limiter protects you from callers, a breaker protects you from callees. It's the subject of the **Design a Rate Limiter** case study (topic 49).

**Next:** [Circuit Breaker, Retry & Bulkhead Patterns](../26-circuit-breaker-retry-and-bulkhead-patterns/why.md) — protecting yourself from the services *you* depend on.
