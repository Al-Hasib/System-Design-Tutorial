# Practice & Interview Questions

**1. What is a cache hit ratio, and why can even a high hit ratio still be a scaling problem?**
The cache hit ratio is the percentage of requests served from cache rather than the origin. Even a 95% hit ratio means 5% of requests still reach the database — at very high total request volumes, that remaining 5% can still be more traffic than the database can handle, so the absolute miss rate matters as much as the percentage.

**2. Explain the cache-aside pattern and one downside of it.**
In cache-aside, the application checks the cache first; on a miss it queries the database, then writes the result into the cache before returning it. A downside is the "cold cache" problem — the first request for any key is always slow, and if many requests miss the same key at once, it can cause a stampede on the database.

**3. When would you choose write-through over write-back?**
Choose write-through when read consistency is critical and you cannot tolerate the cache and database ever being out of sync (e.g., financial balances), and your write volume/latency budget can absorb writing to two systems synchronously. Choose write-back when write throughput and low write latency matter more than the small risk of losing unflushed data if the cache crashes.

**4. What is write-around caching and when is it useful?**
Write-around writes go directly to the database, bypassing the cache; the cache is only populated later on a read (cache-aside style). It's useful for write-heavy, rarely-read data like logs or audit trails, where caching every write would waste cache space on data unlikely to be read again soon.

**5. Describe two cache invalidation strategies and a trade-off of each.**
TTL-based expiration automatically expires entries after a set time — simple and self-healing, but data can be stale for up to the full TTL window. Explicit invalidation actively removes/updates cache entries when the underlying data changes — fresher, but requires every mutation code path to remember to invalidate, and missing one causes hard-to-debug stale-data bugs.

**6. What is a cache stampede, and how would you prevent one?**
A cache stampede (thundering herd) happens when a popular cache key expires and a large number of concurrent requests all miss it at the same instant, hammering the database simultaneously. Mitigations include request coalescing/locking (only one request repopulates the cache while others wait), serving stale data briefly while refreshing in the background, and staggering/jittering TTLs so hot keys don't all expire at once.

**7. Compare LRU, LFU, and FIFO eviction policies.**
LRU evicts the entry that hasn't been accessed in the longest time, favoring recently used data. LFU evicts the entry with the fewest total accesses, favoring consistently popular data over recency. FIFO evicts the oldest inserted entry regardless of access pattern, which is simple but often less effective than LRU or LFU for real-world access patterns.

**8. Your product page cache is set to a 60-second TTL, but customer support says users sometimes see outdated prices right after an admin updates them. How would you fix this?**
Add explicit invalidation: whenever a price is updated, actively delete or refresh that product's cache entry instead of relying solely on the 60-second TTL to expire it. TTL can remain as a safety net, but explicit invalidation removes the staleness window for price-sensitive updates.

**9. Why does write-back caching risk data loss, and how might you reduce that risk?**
Write-back acknowledges a write as soon as it's in the cache, before it's persisted to the database; if the cache crashes before flushing pending writes, that data is lost. Risk can be reduced by backing the cache with a durable write-ahead log or replication, flushing frequently/in small batches, or using a caching system with built-in persistence.

**10. If your cache layer goes down entirely, what should happen to your application, and which caching pattern makes this graceful?**
Ideally the application should degrade gracefully and continue serving requests directly from the database (slower, but functional) rather than fail outright. Cache-aside supports this well because the application already knows how to query the database directly on a cache miss — it just treats a cache outage as a permanent miss.

**11. Why might you deliberately choose a shorter TTL for a "trending" or "hot" content feed versus a static "About Us" page?**
A trending feed changes frequently and users expect near-real-time freshness, so a short TTL (seconds) limits staleness while still absorbing most read traffic. A static About Us page rarely changes, so a long TTL (hours or more) maximizes cache efficiency without any meaningful freshness cost.

**12. What's the difference between a cache miss and stale data, and why does that distinction matter operationally?**
A cache miss means the data simply isn't present in the cache yet, requiring a fetch from the source of truth. Stale data means the cache does contain an entry, but it no longer reflects the current state of the source of truth. This matters because a miss only costs latency, while serving stale data can cause correctness bugs that are much harder to detect and debug.
