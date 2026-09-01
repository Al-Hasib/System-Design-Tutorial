# Practice & Interview Questions

Follow-up questions an interviewer might ask after the main design discussion, with concise model answers.

**1. How would you handle custom aliases requested by users?**
- Treat custom-alias requests as a separate write path from auto-generated codes: check the database (or a bloom filter first, for a fast negative check) for existing use of the requested alias before reserving it.
- If taken, return a conflict error and let the user pick another; if available, write it with the same schema as auto-generated codes so the read path doesn't need to branch.
- This reintroduces the collision problem the counter-based generator avoids, so it's inherently a bit slower and needs a uniqueness constraint or conditional write.

**2. How do you keep hot/viral URLs fast under a sudden traffic spike?**
- Rely on the cache-aside layer: popular links stay resident under LRU eviction because they're re-read constantly, so most of the spike is absorbed by cache, not the database.
- For an extreme spike (a link going viral in minutes), consider a short local in-memory cache on each app server in addition to the shared cache, to shave off network hops entirely for the very hottest keys.
- Consistent hashing across cache nodes ensures the extra load from one hot key doesn't unevenly overload a single shard as the cluster scales.

**3. How would you add click analytics without slowing down redirects?**
- Never do analytics work synchronously on the redirect path. Log a lightweight click event (short code, timestamp, maybe referrer/IP) to a message queue.
- A separate downstream consumer/pipeline aggregates these events into a data store optimized for analytics queries.
- This keeps the redirect's critical path down to a cache lookup plus an HTTP 301/302, with analytics accepted as eventually consistent.

**4. What happens when you run out of short codes (key exhaustion)?**
- With base62 and 7 characters, the key space is about 3.5 trillion codes, comfortably above the 30 billion needed over 5 years, so exhaustion isn't realistic at this scale.
- If it ever became a real concern, the fix is to increase code length (adding one character multiplies the space by 62) rather than redesigning the encoding scheme.
- Reclaiming codes from expired/deleted links is possible but adds complexity (must guarantee the code isn't cached or referenced anywhere) and is rarely worth it given how much headroom base62 already provides.

**5. SQL or NoSQL — which would you actually pick, and why?**
- The core access pattern is a simple key lookup (short code -> long URL), with no joins or complex transactions, which favors a NoSQL key-value store like DynamoDB or Cassandra for easier horizontal scaling.
- A sharded relational database (MySQL/PostgreSQL) is also defensible, especially if custom-alias uniqueness needs a strong constraint, or the team already has relational operational expertise.
- This is ultimately a CAP theorem trade-off: NoSQL options here typically favor availability and partition tolerance (AP) with eventual consistency, which is fine for click counts but requires extra care for alias uniqueness at write time.

**6. How do you protect the system from malicious or duplicate URL submissions?**
- Validate and sanitize submitted URLs (well-formed, not pointing at internal/private IP ranges to prevent SSRF-style abuse).
- Check submissions against a blocklist or threat-intelligence/malware-URL service before shortening, and consider re-checking periodically since a URL can turn malicious after the fact.
- For duplicates, optionally hash the long URL and check if it's already been shortened, returning the existing short code instead of creating a redundant entry — this also naturally reduces storage growth.

**7. How would you support expiring links?**
- Store an optional `expiresAt` timestamp on each record; the read path checks it before redirecting and returns a "link expired/not found" response if it has passed.
- Expired entries can be purged lazily (checked on read) or via a periodic batch job that removes them from the database and evicts them from the cache.
- Cache entries for expired links need either a TTL matching the expiry or active invalidation, otherwise the cache could serve a redirect past its expiration.

**8. How would you deploy this across multiple regions?**
- Deploy app servers, cache, and database shards per region, with a global load balancer or DNS-based routing (e.g., latency-based routing) sending users to their nearest region.
- Replicate the short-code-to-URL mapping across regions, accepting eventual consistency for newly created links (a link created in one region may take a moment to appear in another) in exchange for lower read latency everywhere.
- Consistent hashing helps within each region's cache/database cluster, while cross-region replication is handled separately, typically asynchronously, to avoid coupling write latency in one region to network round-trips to another.

**9. How does cache invalidation work if a custom alias is edited or deleted?**
- Active invalidation is required: on update/delete, the app server explicitly evicts (or overwrites) the corresponding cache entry rather than relying solely on a TTL.
- If the cache is distributed, invalidation needs to reach every node/replica holding that key, not just the one the app server happens to talk to — a pub/sub invalidation message across cache nodes is a common pattern.
- Until invalidation completes, there's a brief window where a stale redirect could be served; this is an accepted trade-off unless the product requirements demand immediate consistency on edits.

**10. Why choose a counter-based key generation service over just hashing the URL?**
- Hash-based generation (e.g., base62-encoding an MD5/SHA digest) needs a collision check-and-retry loop on every write, and retries get more frequent as the table fills.
- A counter-based approach guarantees uniqueness by construction, so there's no retry loop on the write path at all.
- The trade-off is operational: a counter service is a small stateful component that itself needs to scale, typically by sharding the counter (e.g., odd/even ranges, or embedding a region/machine ID Snowflake-style) or by handing out pre-allocated ID batches to each app server.

**11. How would you rate-limit the shorten endpoint to prevent abuse?**
- Apply a token bucket or sliding-window rate limiter per API key, user account, or source IP at the load balancer/API gateway layer, before requests reach app servers.
- Return a 429 Too Many Requests once a client exceeds its quota, with a `Retry-After` header.
- This protects both against key-space exhaustion from scripted abuse and against a malicious actor using the service to mass-generate redirects to harmful destinations.

**12. How would you scale the database as data grows well beyond initial projections?**
- Shard the database using consistent hashing on the short code, so adding new shards only remaps a small fraction of keys rather than triggering a full rebalance.
- Keep shards roughly balanced by choosing a hash function with good key distribution, and use virtual nodes on the hash ring so uneven physical shard capacity doesn't create hot spots.
- Combine sharding with the cache-aside layer so most read traffic never reaches the database at all, keeping each shard's actual query load well below its theoretical capacity.
