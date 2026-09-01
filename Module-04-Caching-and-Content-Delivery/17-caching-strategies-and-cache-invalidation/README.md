# Caching Strategies & Cache Invalidation (Cache-Aside, Write-Through, Write-Back)

**Difficulty:** Intermediate
**Estimated Length:** 12-15 min
**Prerequisites:** [04 - Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md), [12 - Database Indexing Explained](../../Module-03-Databases-and-Storage/12-database-indexing-explained/README.md)

## Learning Objectives

- Explain why caching is often the highest-leverage optimization in system design
- Compare cache-aside, read-through, write-through, write-behind (write-back), and write-around strategies
- Describe common cache invalidation techniques and why "cache invalidation is hard"
- Identify cache eviction policies (LRU, LFU, FIFO) and when to use each
- Recognize and mitigate common cache failure modes: stampede, thundering herd, and stale reads

## Script

### Hook / Intro

There's an old joke in computer science: "There are only two hard things in computer science: cache invalidation and naming things." Today we're tackling the first one. If you've ever refreshed a webpage and seen old data, or wondered how Twitter can show a billion people their feed without melting its database, the answer is almost always caching. In this video we're going to break down the major caching strategies — cache-aside, read-through, write-through, write-behind, and write-around — and then talk about the genuinely hard part: how you invalidate a cache without breaking your users' trust in the data.

### Why Caching Matters

Let's start with the "why." Every time your application needs data, it has two choices: go fetch it from a slow, expensive source — like a database on disk, or a third-party API over the network — or grab it from somewhere fast, if it already computed or fetched that data recently. That "somewhere fast" is a cache. It's usually backed by RAM, and RAM is roughly 100,000 times faster to access than a spinning disk, and still 5-10x faster than even a fast SSD. So if you can serve a request from memory instead of from a database query that hits disk, you can go from tens of milliseconds down to sub-millisecond response times.

But it's not just about speed for the individual user — it's about protecting your entire system. Imagine a product page that gets hit 10,000 times a second. Without a cache, that's 10,000 database queries a second, all asking the same question. With a cache, maybe 1 in every 1,000 requests actually reaches the database, because the other 999 are served from a cached copy. That ratio — the percentage of requests served from cache — is called the **cache hit ratio**, and it is one of the most important numbers in a caching system. A hit ratio of 95% might sound great, but it still means 5% of a huge amount of traffic is reaching your origin, so even "good" cache hit ratios need to be evaluated against your actual scale.

### Cache-Aside (Lazy Loading)

The most common pattern you'll implement yourself is **cache-aside**, also called lazy loading. Here's how it works: when your application needs data, it first checks the cache. If the data is there — a "cache hit" — it returns it immediately. If it's not there — a "cache miss" — the application queries the database itself, gets the result, writes a copy into the cache, and then returns it to the caller. The cache sits "aside" the main data path; it's the application's job to manage it, not the cache's.

The nice thing about cache-aside is resilience: if your cache goes down entirely, your app still works — it just falls back to hitting the database for every request, which is slower but not broken. The downside is the first request for any given piece of data is always slow, since it has to populate the cache. This is called a "cold cache" problem, and at scale, if many requests for the same missing key arrive at once, you get a **cache stampede** — hundreds of threads all missing the cache at the same moment and hammering the database simultaneously. We'll talk about how to guard against that in a minute.

### Read-Through Caching

**Read-through** is a close cousin of cache-aside, but the responsibility shifts. Instead of your application code checking the cache and then manually querying the database on a miss, the cache itself is configured to know how to load data from the database. Your application just always asks the cache, and the cache internally fetches from the source of truth if needed. This keeps your application code cleaner, since the loading logic lives in one place, but it requires a caching layer that supports this — many managed caching services and libraries provide it.

### Write-Through Caching

Now let's talk about writes. In **write-through** caching, every time your application writes data, it writes to the cache and the database at the same time, as a single logical operation, before returning success to the caller. This guarantees the cache and the database are always in sync — you'll never read stale data from the cache after a write completes. The cost is write latency: every write now has to wait on two systems instead of one. Write-through is a good choice when read consistency matters a lot and your write volume is moderate.

### Write-Back (Write-Behind) Caching

**Write-back**, sometimes called write-behind, takes the opposite trade-off. The application writes only to the cache, and the cache acknowledges success immediately. Separately, asynchronously, the cache flushes those writes to the database in the background — maybe batched, maybe on a timer. This makes writes extremely fast, because you're not waiting on the slow database at all. But it introduces risk: if the cache crashes before it flushes pending writes to the database, that data is gone. Write-back is used when write throughput is critical and you can tolerate a small window of potential data loss, or you've built durability into the cache layer itself, for example with an append-only log.

### Write-Around Caching

Last pattern: **write-around**. Here, writes go directly to the database, bypassing the cache entirely. The cache is only populated later, on a subsequent read, using the cache-aside pattern. This avoids filling the cache with data that might never be read again — think of a system logging billions of write-once, read-rarely events. The trade-off is that a read immediately after a write will always be a cache miss.

### Cache Invalidation

Now, the hard part: keeping cached data from going stale. There are three broad approaches. **Time-based expiration (TTL)** is the simplest — you set every cache entry to automatically expire after some number of seconds. It's simple and self-healing, but during that TTL window, users might see outdated data. **Explicit invalidation** is more precise: whenever the underlying data changes, your application actively deletes or updates the corresponding cache entry. This gives you fresher data but adds complexity, because now every code path that mutates data has to remember to also touch the cache — miss one, and you get a stale-data bug that's painful to track down. **Write-through invalidation**, which we already covered, sidesteps the problem by keeping the cache updated as part of every write.

In practice, most production systems use a mix: TTLs as a safety net, plus explicit invalidation for anything user-facing where staleness is unacceptable, like an account balance.

### Eviction Policies

Caches have limited memory, so when they're full, something has to be evicted to make room for new entries. The three classic policies are: **LRU (Least Recently Used)**, which evicts whatever hasn't been accessed in the longest time — great for most workloads because recently accessed data tends to be accessed again soon; **LFU (Least Frequently Used)**, which evicts whatever has been accessed the fewest total times — better when popularity is more stable than recency; and **FIFO (First In, First Out)**, which simply evicts the oldest entry regardless of how often it's used — simple but often less effective. Redis, for example, lets you configure which of several eviction policies to use, including LRU and LFU variants.

### Real-World Example

Think about a news website's homepage. It's read millions of times a minute but only updated by editors a handful of times an hour. That's a perfect cache-aside plus TTL candidate: cache the rendered homepage for, say, 30 seconds. Nearly every visitor gets an instant response from cache, and worst case, an update takes 30 seconds to show up everywhere — a totally acceptable trade-off. Now compare that to a shopping cart: if a user adds an item, they expect to see it immediately, every time, everywhere they look. That calls for write-through or explicit invalidation, not a lazy TTL, because staleness there directly breaks the user experience.

One more failure mode worth knowing: the **thundering herd** or cache stampede problem I mentioned earlier. Imagine a hot cache key — say, a celebrity's profile — expires at the exact same moment ten thousand requests arrive. All ten thousand miss the cache and slam the database at once. A common fix is "cache locking" or "request coalescing": the first request to miss acquires a lock, fetches the data, and populates the cache, while the other 9,999 requests either wait briefly or get served slightly stale data instead of also hitting the database.

### Recap

Let's recap. Cache-aside is the default, application-managed pattern where you check the cache first and populate it on a miss. Read-through moves that logic into the cache itself. Write-through keeps the cache and database in sync on every write, at the cost of write latency. Write-back is fast but risks data loss. Write-around avoids polluting the cache with rarely-read writes. And no matter which pattern you use, you need a plan for invalidation — TTLs, explicit invalidation, or both — plus an eviction policy like LRU for when the cache fills up.

### What's Next

Caching so far has been about caching data close to your application server. But what about caching content close to your *users*, wherever in the world they are? In the next video, we'll look at Content Delivery Networks — CDNs — and how services like Cloudflare and CloudFront push your content to edge servers around the globe so a user in Tokyo doesn't have to wait on a server in Virginia.

## Key Takeaways

- Caching trades a small amount of staleness risk for a massive reduction in latency and origin load; RAM access is orders of magnitude faster than disk.
- Cache-aside (lazy loading) is application-managed: check cache, on miss query the database and populate the cache.
- Write-through keeps cache and DB consistent on every write but adds write latency; write-back is fast but risks losing unflushed writes; write-around avoids caching rarely-read writes.
- Invalidation strategies: TTL-based expiration (simple, self-healing, but stale during the window) and explicit invalidation (fresh, but must be applied consistently everywhere data changes).
- Eviction policies (LRU, LFU, FIFO) determine what gets removed when the cache is full; LRU is the most common default.
- Cache stampede/thundering herd occurs when a hot key expires and many requests hit the database at once; mitigate with locking, request coalescing, or staggered TTLs.
