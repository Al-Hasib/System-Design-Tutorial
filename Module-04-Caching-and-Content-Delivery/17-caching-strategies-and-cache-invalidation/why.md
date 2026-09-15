# Why This Topic Matters: Caching Strategies & Cache Invalidation

> **In one sentence:** Caching is the cheapest order-of-magnitude performance win available, and cache invalidation is the reason it's also one of the most common sources of bugs that are impossible to reproduce.

## The World Before This Idea

Every request recomputes everything. A product page runs the same six queries for the same product, 4,000 times a second, returning byte-identical results. The database does the same work over and over because nobody told it not to.

Your options at that point are to buy a bigger database, shard, or stop asking. Caching is "stop asking" — and it's dramatically cheaper than the other two. A memory read is measured in microseconds; a disk-backed database query with a network hop is measured in milliseconds. That's a 100-1000x gap, available for the price of a Redis instance.

## The Problems It Solves

### 1. Repeated identical work
**What you see:** Database CPU pinned, with a query log showing the same handful of queries dominating volume.

**Why it happens:** Read-heavy workloads are extremely repetitive. A small set of popular items usually accounts for a large majority of traffic.

**How caching solves it:** Compute once, serve many. A 95% hit rate means the database sees 5% of the load — turning a database you were about to shard into one that's comfortably idle.

### 2. Latency floors set by the data layer
**What you see:** A page can't get below 200 ms because it makes several database calls, each unavoidably tens of milliseconds.

**Why it happens:** Disk I/O, network hops, and query planning have real costs that no amount of indexing removes entirely.

**How caching solves it:** An in-memory lookup is sub-millisecond. Caching is often the only way to hit aggressive latency targets on data that isn't trivially cheap to compute.

### 3. Stale data users can see
**What you see:** A user updates their profile and the old version keeps appearing. A price change doesn't take effect for an hour. Two users see different values for the same thing.

**Why it happens:** A cache is a second copy of the truth. The moment the source changes, the copy is wrong, and nothing automatically tells the cache.

**How invalidation strategy solves it:** This is the core of the topic. TTLs bound staleness cheaply but imprecisely. Explicit invalidation on write is precise but requires every write path to remember — and the one that forgets is the one that bites you. Write-through keeps the cache correct at the cost of write latency. Each strategy trades freshness against complexity, and choosing consciously is the whole game.

### 4. Cache failure taking down the database
**What you see:** The cache restarts. Every request misses simultaneously, all of them hit the database at once, and the database — sized for 5% of traffic — dies instantly.

**Why it happens:** A cache doesn't just speed things up; it becomes a load-bearing part of your capacity plan. Removing it doesn't return you to the old world, it drops full traffic onto a system that was scaled down to match.

**How this topic solves it:** It names the failure modes and their remedies. **Thundering herd / stampede** — many requests missing the same key at once — is handled with request coalescing or a short lock. **Cache penetration** — repeated misses for keys that don't exist — is handled by caching negative results or a Bloom filter. **Cache avalanche** — many keys expiring together — is handled by jittering TTLs.

## The Price You Pay

- **A second source of truth.** Every cached value can be wrong. You've traded a correctness guarantee for speed, and that trade must be acceptable for that specific data.
- **Invalidation is genuinely hard.** It's a cliché because it's true: knowing every code path that invalidates a given cached value, forever, as the codebase grows, is a real maintenance burden.
- **Debugging gets much harder.** "It works for me" often means "my request hit a different cache node" or "my key expired." Bugs become intermittent and environment-dependent.
- **New failure modes.** Stampedes, avalanches, and penetration are problems you did not have before.
- **Caching the wrong things wastes everything.** Data with low reuse or high churn gets evicted before it's read twice — you pay all the complexity and get no hit rate.

## When You Need It — and When You Don't

| Cache when | Don't cache when |
|---|---|
| Reads vastly outnumber writes | Data changes on nearly every read |
| The same data is requested repeatedly | Every request is unique (per-user one-offs) |
| Computing the value is expensive | The source query is already sub-millisecond |
| Slightly stale data is acceptable | Correctness must be exact (balances, inventory at checkout) |

## Why This Shows Up in Interviews

Caching appears in virtually every design that has read traffic, so *mentioning* it is table stakes. Credit comes from the follow-ups: which strategy (cache-aside is the sensible default, and you should know why), what's the TTL and why, how do you invalidate on write, what's your expected hit rate, and — the question that separates candidates — what happens when the cache goes down or cold-starts. Talking about stampede protection unprompted signals real operational experience.

## How It Connects

Caching is the first line of defense that makes **replication** and **sharding** unnecessary for many systems. It's implemented at multiple layers: the browser and **CDN** (topic 18) at the edge, a **reverse proxy** in the middle, and **Redis/Memcached** (topic 19) behind your services. Distributing a cache across nodes needs **consistent hashing**. And the freshness trade-off is a direct application of **eventual consistency** — a cache is, formally, an eventually consistent replica you built on purpose.

**Next:** [CDN Explained](../18-cdn-explained/why.md) — caching moved as close to the user as physically possible.
