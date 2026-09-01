# Design a URL Shortener

**Difficulty:** Advanced (Capstone)
**Estimated length:** 20-30 min
**Prerequisites:**
[Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md),
[Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md),
[SQL vs NoSQL](../../Module-03-Databases-and-Storage/11-sql-vs-nosql/README.md),
[Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md),
[Caching Strategies and Cache Invalidation](../../Module-04-Caching-and-Content-Delivery/17-caching-strategies-and-cache-invalidation/README.md),
[Consistent Hashing Explained](../../Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/README.md),
[Rate Limiting Algorithms](../../Module-06-Distributed-Systems-Concepts/25-rate-limiting-algorithms/README.md)

## Learning Objectives

- Run a complete mock interview for a classic system design question end-to-end, from clarifying requirements to trade-off discussion.
- Practice back-of-the-envelope capacity estimation (QPS, storage, bandwidth) under interview time pressure.
- Apply consistent hashing, cache-aside caching, and database sharding — concepts from earlier modules — to a single, concrete system.
- Compare counter-based and hash-based key generation strategies and justify a choice.
- Articulate the trade-offs between SQL and NoSQL storage, and between strong and eventual consistency, in the context of a real feature (custom aliases).

## Script

### Hook / Intro

"Design a URL shortener" is probably the single most common system design interview question — which is exactly why it's the right capstone for this course. It looks simple on the surface: take a long URL, give back a short one, and redirect people when they click it. But underneath that simple ask is almost every idea we've covered in this series — capacity estimation, load balancing, caching, sharding, consistent hashing, and rate limiting. In this video I'm going to run this exactly the way I'd run it in a real interview: clarify requirements, estimate scale, sketch the high-level design, dive deep into two or three components, and then spend real time on trade-offs. If you've watched the rest of this module, you'll recognize almost every piece we use here — that's the point.

### Step 1: Clarify Requirements

Before drawing a single box, I clarify scope with the interviewer. I'd ask questions like: What's our expected scale? Do we need custom, user-chosen aliases, or only auto-generated short codes? Do links expire? Do we need click analytics? Is this a single global service or does it need multi-region deployment?

For this session, let's lock in the following.

**Functional requirements:**
- Given a long URL, generate a unique, short alias (e.g. `short.ly/aZ9kQ2`).
- Given a short alias, redirect the user to the original long URL (HTTP 301/302).
- Support optional custom aliases chosen by the user.
- Support optional expiration dates on links.
- Basic click analytics (count and timestamp) are a nice-to-have, not core.

**Non-functional requirements:**
- High availability — a broken redirect service breaks every link ever shared, so uptime matters more than perfect consistency here.
- Low latency on redirects — the read path should feel instant, ideally single-digit milliseconds from cache.
- The system should be highly scalable in reads, since links get shared and clicked far more often than they're created.
- Short codes must not collide, and ideally shouldn't be easily guessable/enumerable.

That last non-functional point already tells us something important: this is a read-heavy system, so most of our design energy should go into making the read (redirect) path fast and cheap, while the write (shorten) path can afford to be a bit heavier.

### Step 2: Capacity Estimation

Let's put real numbers on this, because "web scale" isn't a number an interviewer will accept.

**Assumptions:**
- 500 million new URLs written per month.
- A 100:1 read-to-write ratio (each link is clicked, on average, 100 times).
- Each URL record is roughly 500 bytes (long URL, short code, metadata, timestamps, owner ID).
- Data must be retained for 5 years.

**Traffic:**
- Reads per month = 500M × 100 = 50 billion redirects/month.
- Write QPS (average) = 500,000,000 / (30 × 24 × 3600 seconds) ≈ 500,000,000 / 2,592,000 ≈ **~193 writes/sec**.
- Read QPS (average) = 50,000,000,000 / 2,592,000 ≈ **~19,300 reads/sec**.
- Traffic is rarely uniform, so I'd design for a peak multiplier of roughly 2-3x average — call it **~40,000-60,000 reads/sec at peak**. That peak number is what actually drives our caching and load balancing decisions.

**Storage (5 years):**
- Total records = 500M/month × 12 months × 5 years = **30 billion URLs**.
- Total storage = 30,000,000,000 × 500 bytes = 15,000,000,000,000 bytes = **~15 TB** of raw metadata over 5 years. That's easily shardable across a modest cluster of commodity database nodes.

**Bandwidth:**
- Write bandwidth = 193 writes/sec × 500 bytes ≈ ~96 KB/sec — trivial.
- Read bandwidth = 19,300 reads/sec × ~500 bytes (redirect response) ≈ ~9.6 MB/sec average, and roughly 20-30 MB/sec at peak — still modest, but it confirms reads dominate the system's footprint, not storage.

**Key space check:** if we encode short codes in base62 (`[a-zA-Z0-9]`) with 7 characters, we get 62^7 ≈ 3.5 trillion possible codes — comfortably more than the 30 billion URLs we project over 5 years, with huge headroom for growth.

The takeaway from this section: writes are light (a couple hundred per second), reads are heavy (tens of thousands per second, bursting higher), and storage, while large in absolute terms, is small enough that a single well-chosen database technology with sharding will handle it comfortably. This read-heavy, storage-light profile is what shapes every decision from here on.

### Step 3: High-Level Design

Given that profile, here's the shape of the system:

```
Client -> Load Balancer -> App Servers (stateless) -> Cache -> Database
                                    |
                          Key Generation Service
```

- **Client** issues two kinds of requests: `POST /shorten` with a long URL (and optionally a custom alias/expiry), and `GET /{shortCode}` to be redirected.
- **Load balancer** sits in front of a fleet of stateless application servers, distributing traffic and giving us horizontal scalability and failover — this is straight out of the load balancing fundamentals we covered earlier: round-robin or least-connections routing across app servers, with health checks removing dead nodes.
- **App servers** are stateless — any server can handle any request — which is exactly what lets us scale horizontally by just adding more boxes behind the load balancer.
- **Key Generation Service** hands out unique short codes to the write path, decoupled from the app servers so key allocation doesn't become a bottleneck or a source of collisions.
- **Cache** sits in front of the database on the read path — since reads outnumber writes 100:1, a cache absorbs the overwhelming majority of redirect traffic.
- **Database** is the durable source of truth for the mapping between short code and long URL, plus metadata (creation time, expiry, owner, click count).

The write path: client submits a long URL, an app server asks the key generation service for a unique code (or validates a requested custom alias), writes the mapping to the database, populates the cache, and returns the short URL.

The read path: client hits `GET /{shortCode}`, the app server checks the cache first; on a hit, it issues the redirect immediately; on a miss, it reads from the database, populates the cache, and then redirects.

### Step 4: Deep Dive on Key Components

**4a. Key generation: counter-based vs. hash-based.** There are two classic approaches. The first is hash-based: run the long URL through MD5 or SHA-256, base62-encode the first 7 characters of the digest, and use that as the short code. It's simple and stateless, but collisions are possible, so you need a check-and-retry loop against the database, which adds write-path latency as the table fills up. The second approach is counter-based: maintain a globally unique, monotonically increasing counter (backed by something like a dedicated key-generation service that hands out pre-allocated ranges of IDs to each app server), and base62-encode the counter value into a short code. This guarantees no collisions and no retries, at the cost of running a small stateful service. In an interview, I'd propose the counter-based approach specifically because it removes collision handling from the hot write path entirely — each app server can request a batch of, say, 1,000 IDs at a time and hand them out locally, which also cuts down on round trips to the key service.

**4b. Caching the read path — this is where cache-aside from Module 4 comes in directly.** Because redirects are 100x more frequent than writes, and because real-world link popularity follows a heavy power-law (a small fraction of links account for most clicks), a cache-aside strategy is the natural fit: the app server checks the cache first, and only falls through to the database on a miss, then populates the cache with what it just read. We'd use an LRU eviction policy so that hot links stay resident and cold ones get pushed out automatically. Given ~19,300 average reads/sec and a heavy-tailed access pattern, caching even the top 20% of links by popularity would absorb the large majority of traffic, keeping database load far below the peak read number we calculated earlier.

**4c. Scaling the database — this is where sharding and consistent hashing from Modules 3 and 6 come in.** At 30 billion rows over 5 years, a single database instance won't hold or serve this comfortably, so we shard by short code. The naive approach — `hash(shortCode) % N` — works until you add or remove a shard, at which point almost every key remaps and you trigger a massive, unnecessary data migration. That's precisely the problem consistent hashing solves: shards (and cache nodes, for that matter) sit on a hash ring, and adding or removing a node only remaps the keys immediately adjacent to it on the ring, not the entire keyspace. I'd apply consistent hashing at two layers here — across our cache nodes (e.g., a Redis cluster) and across our database shards — so that scaling the cluster up or down as traffic grows doesn't cause a cache-wide or database-wide stampede.

**4d. Rate limiting the write path.** Since anyone can call `POST /shorten`, we need to defend against abuse — someone scripting mass URL creation to spam or exhaust our key space. This is exactly the rate limiting problem from Module 6: a token bucket or sliding-window limiter per API key or per source IP, enforced at the load balancer or API gateway layer before a request ever reaches an app server.

### Step 5: Bottlenecks & Trade-offs

No design is complete without naming what we gave up.

- **Custom aliases vs. auto-generated codes:** custom aliases are great for UX and branding, but they reintroduce the collision problem the counter-based generator was designed to avoid — a requested alias might already be taken, requiring a uniqueness check against the database on the write path. I'd keep this as a separate, slightly slower code path from the default auto-generated flow, with a simple existence check plus a reservation write.
- **SQL vs. NoSQL:** the access pattern here — simple key lookups by short code, no complex joins or transactions — is a textbook fit for a NoSQL key-value or wide-column store (like DynamoDB or Cassandra), which scales horizontally more naturally than a relational database. That said, a well-sharded relational database (MySQL/PostgreSQL) is a perfectly defensible choice too, especially if the team already runs one and wants stronger consistency guarantees for things like uniqueness constraints on custom aliases. This is a real CAP theorem trade-off: a NoSQL store optimized for availability and partition tolerance gives us eventual consistency on things like click counts, which is fine for analytics but requires more care for alias uniqueness.
- **Cache invalidation on custom aliases:** if a user is later allowed to edit or delete a custom alias, the cache entry for the old code must be actively invalidated, not just left to expire — otherwise the system keeps redirecting to stale or deleted content until the TTL lapses. This is the classic "cache invalidation is one of the two hard things" problem, and it's worth calling out explicitly.
- **ID/key generation at scale:** a single centralized counter is a single point of contention and failure. In practice, you'd shard the counter itself (e.g., odd/even ranges per data center, or Snowflake-style IDs embedding a machine/region identifier) so no one component becomes a bottleneck as write throughput grows.
- **Analytics vs. latency:** incrementing a click counter synchronously on every redirect would slow down the hot path. The right trade-off is to log click events asynchronously (e.g., to a queue) and aggregate them in a separate analytics pipeline, keeping the redirect itself on the fast, cache-only path.

### Recap

We started from two numbers — 500M writes/month and a 100:1 read ratio — and let them drive every decision: a stateless app tier behind a load balancer, a counter-based key generation service to avoid collisions, a cache-aside layer to absorb the read-heavy traffic, and consistent-hashed sharding across both the cache and the database to scale without painful rebalancing. Along the way we made explicit trade-offs on custom aliases, SQL vs. NoSQL, cache invalidation, and ID generation — the kind of trade-off discussion that's usually worth more in an interview than the diagram itself.

### What's Next

This wraps the case-studies module. If you want to keep sharpening these skills, go back through the earlier deep-dive videos on consistent hashing, sharding, and caching strategies with this URL shortener design in mind — you'll notice how often the same handful of primitives get reused across completely different systems, whether that's a chat app, a news feed, or a payments platform.

## Key Takeaways

- Always clarify functional and non-functional requirements before designing — they determine whether you're optimizing for reads, writes, or consistency.
- Back-of-the-envelope math (QPS, storage, bandwidth) should drive architectural decisions, not just decorate the interview.
- This system is read-heavy (100:1), which is why cache-aside caching and horizontal read scaling matter more here than write optimization.
- Counter-based key generation avoids collisions on the hot write path; hash-based generation is simpler but needs collision retries.
- Consistent hashing minimizes data movement when scaling cache or database nodes, versus naive modulo-based sharding.
- Trade-off discussions — SQL vs. NoSQL, cache invalidation, custom aliases, CAP theorem implications — are where interviews are actually won or lost.
