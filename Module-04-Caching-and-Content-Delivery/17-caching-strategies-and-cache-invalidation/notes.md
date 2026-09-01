# Study Notes: Caching Strategies & Cache Invalidation

## Definitions

- **Cache**: A fast, typically memory-backed storage layer that holds copies of data so future requests can be served without repeating expensive computation or I/O.
- **Cache hit**: A request served directly from the cache.
- **Cache miss**: A request that is not found in the cache and must fall back to the source of truth (database, API, computation).
- **Cache hit ratio**: hits / (hits + misses). Higher is generally better, but must be evaluated relative to absolute traffic volume.
- **TTL (Time To Live)**: The duration a cache entry is considered valid before it automatically expires.
- **Stale data**: Cached data that no longer matches the current state of the source of truth.
- **Cache stampede / thundering herd**: Many concurrent requests missing the cache at once (e.g., after a hot key expires) and overwhelming the origin.

## Caching Strategies Comparison

| Strategy | Who manages it | Write path | Read path | Consistency | Best for |
|---|---|---|---|---|---|
| Cache-Aside (Lazy Loading) | Application | App writes to DB; cache updated/invalidated separately (or not at all) | App checks cache first, on miss reads DB then populates cache | Eventually consistent (until TTL/invalidation) | General-purpose, read-heavy workloads |
| Read-Through | Cache library/service | Same as cache-aside or paired with write-through | App always asks cache; cache loads from DB internally on miss | Same as cache-aside | Simplifying app code when supported by the cache library |
| Write-Through | Cache + application (synchronous) | Write goes to cache and DB together, synchronously | Cache always has fresh data | Strong (cache never stale after a completed write) | Read-heavy data where correctness matters, moderate write volume |
| Write-Back (Write-Behind) | Cache (asynchronous) | Write goes to cache only; cache flushes to DB later, async | Fast, fresh in cache | Cache is fresh; DB may lag; risk of data loss on cache crash | Write-heavy workloads needing low write latency |
| Write-Around | Database directly | Write goes straight to DB, bypassing cache | First read after write is a cache miss (cache-aside populates it) | Avoids cache pollution from rarely-read writes | Write-once/read-rarely data (e.g., logs, audit trails) |

## Cache Invalidation Approaches

| Approach | How it works | Pros | Cons |
|---|---|---|---|
| TTL / Expiration | Entry auto-expires after N seconds | Simple, self-healing, no extra code paths | Staleness window; picking the right TTL is a balancing act |
| Explicit Invalidation | App deletes/updates cache entry whenever underlying data changes | Freshness on demand | Must be applied on every mutation path; easy to miss one and cause bugs |
| Write-Through (as invalidation) | Cache updated synchronously with every write | No staleness | Added write latency |

## Eviction Policies

| Policy | Evicts | Notes |
|---|---|---|
| LRU (Least Recently Used) | Entry not accessed for the longest time | Most common default; good general-purpose heuristic |
| LFU (Least Frequently Used) | Entry accessed the fewest total times | Better when popularity is stable over time rather than recency-driven |
| FIFO (First In, First Out) | Oldest inserted entry, regardless of usage | Simplest, often less effective than LRU/LFU |

## Key Numbers / Rules of Thumb

- RAM access latency: ~100 nanoseconds; SSD: ~100 microseconds; HDD: ~10 milliseconds — RAM can be ~100,000x faster than disk.
- A well-tuned cache-aside system can often push cache hit ratios above 90-95% for read-heavy workloads.
- Even at a 95% hit ratio, 5% of requests still hit the origin — at high scale (e.g., 10,000 req/s) that's still 500 req/s reaching the database.
- Typical web-page/API TTLs range from a few seconds (hot, frequently changing data) to hours (mostly static data).

## Summary

- Caching is high-leverage: big latency and load wins for relatively low implementation cost.
- Pick a strategy based on your read/write ratio and your tolerance for staleness vs. write latency vs. data-loss risk.
- Always pair a caching strategy with an explicit invalidation and eviction plan — an un-invalidated cache is a bug generator.
- Guard against cache stampedes on hot keys using locks, request coalescing, or jittered TTLs.
