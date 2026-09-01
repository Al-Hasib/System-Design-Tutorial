# Interview Cheat-Sheet: URL Shortener

Quick-reference companion to `README.md`. Use this for a fast pre-interview refresh.

## Requirements

**Functional**
- Shorten a long URL into a unique short alias.
- Redirect a short alias to its original long URL (HTTP 301/302).
- Support optional custom (user-chosen) aliases.
- Support optional link expiration.
- (Nice-to-have) Basic click analytics: count + timestamp.

**Non-functional**
- High availability (a broken redirect breaks every shared link).
- Low-latency redirects (single-digit ms from cache).
- Highly scalable reads (read-heavy workload).
- Short codes must be unique and hard to enumerate/guess.

## Capacity Estimation

| Metric | Assumption / Calculation | Result |
|---|---|---|
| New URLs written | 500M / month | 500,000,000 |
| Read:write ratio | 100:1 | — |
| Redirects (reads) | 500M × 100 | 50B / month |
| Write QPS (avg) | 500M / 2,592,000 sec | ~193 writes/sec |
| Read QPS (avg) | 50B / 2,592,000 sec | ~19,300 reads/sec |
| Read QPS (peak, 2-3x avg) | ~19,300 × 2-3 | ~40,000-60,000 reads/sec |
| Total records (5 yrs) | 500M × 12 × 5 | 30B records |
| Storage (5 yrs) | 30B × 500 bytes/record | ~15 TB |
| Write bandwidth | 193/sec × 500 bytes | ~96 KB/sec |
| Read bandwidth (avg) | 19,300/sec × 500 bytes | ~9.6 MB/sec |
| Key space (base62, 7 chars) | 62^7 | ~3.5 trillion codes |

**Conclusion:** Read-heavy (100:1), storage is large in total but manageable with sharding, writes are light. Design priority: fast cached reads > write throughput.

## High-Level Architecture

```
Client -> Load Balancer -> App Servers (stateless) -> Cache -> Database
                                  |
                        Key Generation Service
```

- **Load balancer**: distributes traffic across stateless app servers; health checks remove dead nodes.
- **App servers**: stateless, horizontally scalable, handle both shorten (write) and redirect (read) requests.
- **Key Generation Service**: hands out unique short codes (counter-based, pre-allocated ID ranges per server).
- **Cache**: cache-aside layer (e.g., Redis) in front of the DB; absorbs the majority of read traffic due to heavy-tailed link popularity; LRU eviction.
- **Database**: durable store of short-code -> long-URL mapping + metadata; sharded using consistent hashing.

**Write path:** client -> app server -> key gen service (get code) -> write to DB -> populate cache -> return short URL.

**Read path:** client -> app server -> cache lookup -> (hit: redirect immediately) / (miss: read DB -> populate cache -> redirect).

## Key Design Decisions & Trade-offs

| Decision | Option A | Option B | Recommendation / Trade-off |
|---|---|---|---|
| Key generation | Hash-based (MD5/SHA + base62) | Counter-based (pre-allocated ID ranges) | Counter-based avoids collision retries on hot write path |
| Storage | SQL (MySQL/PostgreSQL, sharded) | NoSQL (DynamoDB/Cassandra) | NoSQL fits simple key-value access pattern & scales horizontally; SQL fine if strong uniqueness constraints matter |
| Consistency | Strong consistency | Eventual consistency (CAP: prefer AP) | Eventual consistency acceptable for click counts; custom alias uniqueness needs stronger guarantee at write time |
| Sharding strategy | `hash(key) % N` | Consistent hashing | Consistent hashing minimizes remapped keys when nodes are added/removed |
| Custom aliases | Always allow | Auto-generate only | Custom aliases require uniqueness check (extra write-path cost) |
| Cache invalidation | TTL-only | Active invalidation on update/delete | Active invalidation needed when aliases can be edited/deleted, to avoid stale redirects |
| Click analytics | Synchronous increment | Async event logging (queue + pipeline) | Async keeps redirect path fast; analytics aggregated separately |
| Rate limiting | None | Token bucket / sliding window per API key or IP | Needed on write path (`POST /shorten`) to prevent abuse and key-space exhaustion |

## Terminology Used Consistently

Sharding, consistent hashing, load balancer, cache-aside, CAP theorem, base62 encoding, LRU eviction, stateless app servers, rate limiting (token bucket / sliding window).
