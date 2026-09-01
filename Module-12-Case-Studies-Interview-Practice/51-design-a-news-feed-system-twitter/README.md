# Design a News Feed System (like Twitter/Facebook)

**Difficulty:** Advanced (Capstone)
**Estimated length:** 25-30 min
**Prerequisites:**
- [Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md)
- [Caching Strategies and Cache Invalidation](../../Module-04-Caching-and-Content-Delivery/17-caching-strategies-and-cache-invalidation/README.md)
- [Distributed Caching: Redis and Memcached](../../Module-04-Caching-and-Content-Delivery/19-distributed-caching-redis-and-memcached/README.md)
- [Message Queues: Kafka vs RabbitMQ](../../Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/README.md)
- [Publish-Subscribe Pattern](../../Module-05-Messaging-and-Asynchronous-Systems/21-publish-subscribe-pattern/README.md)
- [Batch vs Stream Processing](../../Module-05-Messaging-and-Asynchronous-Systems/23-batch-vs-stream-processing/README.md)
- [Consistent Hashing Explained](../../Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/README.md)

## Learning Objectives

- Run a complete mock interview for a "design a news feed" prompt, from clarifying questions to a defensible high-level architecture.
- Quantify the scale of a Twitter/Facebook-style feed system: daily active users, posts/sec, and fan-out writes/sec.
- Compare fan-out-on-write, fan-out-on-read, and hybrid delivery strategies, and justify which one to use for celebrity accounts.
- Apply consistent hashing, sharding, and caching patterns from earlier modules to solve the hot-key/celebrity problem.
- Articulate the trade-offs between write cost, read latency, and eventual consistency in a real feed system.

## Script

### Hook / Intro

"Design a news feed" is one of the most common capstone questions at companies like Twitter, Facebook/Meta, LinkedIn, and Instagram — and it's popular precisely because it forces you to combine almost everything from a system design course into one answer: sharded storage, caching, pub-sub messaging, and distributed systems trade-offs like consistency versus latency.

In this video we'll run it as a real interview would go: clarify requirements, size the system with real numbers, sketch a high-level architecture, then go deep on the two or three components that actually matter — fan-out strategy and the celebrity problem. We will explicitly reuse ideas from earlier modules in this course: sharding from Module 3, caching from Module 4, message queues and pub-sub from Module 5, and consistent hashing from Module 6. If you haven't watched those, this is exactly the kind of question where they pay off.

### Step 1: Clarify Requirements

Never start designing before you clarify scope. Here's what I'd ask, and the answers I'll assume for this session.

**Functional requirements:**
- Users can create posts (text, optionally media) — this is the write path.
- Users can follow/unfollow other users — a follow graph that's directed and asymmetric (I can follow you without you following me back).
- Users have a home timeline (the "news feed") showing posts from everyone they follow, reverse-chronological or ranked.
- The feed is ranked, not purely chronological — recency, engagement signals, and affinity to the poster all factor in.

I'll explicitly scope out: comments, likes, direct messages, and ad injection — those are separate systems that could each be their own interview question.

**Non-functional requirements:**
- **Low read latency** for loading the home timeline — users expect it to open in well under a second. Reads vastly outnumber writes, so we optimize for read latency at the cost of write complexity.
- **Eventual consistency is acceptable.** If a new post takes a few seconds to show up in a follower's feed, that's fine. We are explicitly not building a strongly consistent system.
- **High write fan-out** must be handled gracefully. A single post from a celebrity account can need to be delivered to tens of millions of followers — this is the crux of the problem and where most of the interview signal comes from.
- The system must be highly available and horizontally scalable; availability is favored over consistency (an AP-leaning system in CAP terms).

### Step 2: Capacity Estimation

Let's put real numbers on this so our architecture decisions are grounded.

- **300 million daily active users (DAU).**
- Average user posts twice a day → **600 million posts/day**, which is about **~7,000 posts/sec average**, and let's assume peak traffic is 3x average, so **~21,000 posts/sec at peak**.
- Average follower count is **500 followers per user** — that's the "normal" case.
- But there's a long tail: roughly **1 million celebrity/influencer accounts have 10 million+ followers each**, some with 50-100 million.
- Read-to-write ratio for a feed system is typically **100:1 to 1000:1** — each user checks their feed far more often than they post. At 300M DAU checking their feed ~10 times/day, that's **3 billion feed reads/day**, or roughly **35,000 reads/sec average**, spiking higher at peak.
- **Fan-out writes/sec**: if we naively fan out every post to every follower's timeline at write time, average case is 7,000 posts/sec × 500 followers = **3.5 million fan-out writes/sec** just from ordinary users. A single celebrity post with 50 million followers alone generates 50 million fan-out writes in a burst — that one event can dwarf the steady-state load system-wide.
- **Storage for precomputed timeline cache**: if we cache the last ~800 post IDs per user's home timeline (a post ID is ~8 bytes, plus metadata ~ let's budget 100 bytes/entry), that's 300M users × 800 entries × 100 bytes ≈ **24 TB** of cache data. That's a lot, and it tells us immediately we need a distributed cache cluster, not a single Redis box.

These numbers already tell us two things: fan-out is the dominant write cost, and we need a caching layer sized in the tens of terabytes, sharded across many nodes.

### Step 3: High-Level Design

At a high level, the request flow looks like this:

**Client → Load Balancer → Post Service / Feed Service.**

- The **Post Service** handles post creation: it validates the post, writes it to a **sharded post store** (a database partitioned by, say, user ID or post ID using consistent hashing — straight from Module 3 and Module 6), and then publishes a "new post" event onto a **message queue / pub-sub system** (Kafka is the natural fit here, per Module 5).
- A **Fan-out Service** subscribes to that event stream. For each new post, it looks up the author's followers in the **follow-graph store** and decides how to deliver the post: either push it into followers' precomputed timelines immediately (fan-out-on-write), or do nothing and let it be pulled later (fan-out-on-read) — more on this in the deep dive.
- A **Ranking Service** scores candidate posts (recency, affinity, predicted engagement) either at fan-out time or at read time, depending on the strategy.
- A **Cache layer** (Redis cluster, sharded via consistent hashing) stores precomputed home timelines as sorted sets keyed by user ID, so the **Feed Service** can serve a "get my timeline" request with a handful of cache reads instead of a fan-out query across the whole follow graph.
- The **Follow-Graph Store** is a separate service/database optimized for graph queries like "who does user X follow" and "who follows user Y" — this needs its own indexing strategy since follower lists for celebrities are huge.

So the read path is: Client → LB → Feed Service → Redis (precomputed timeline) → hydrate post content from post store/CDN → rank/merge → return. The write path is: Client → LB → Post Service → sharded post DB + publish event → Fan-out Service (via queue) → update caches.

### Step 4: Deep Dive on Key Components

**4a. Fan-out-on-write vs. fan-out-on-read (Module 5: queues and pub-sub).**
Fan-out-on-write (push model) means: when a post is created, we immediately push a reference to it into every follower's precomputed timeline cache. Reads become cheap — just fetch the cached list. But writes become expensive and bursty, exactly the problem we saw with celebrity accounts generating tens of millions of fan-out operations. We use a Kafka topic per shard of the follower list so the fan-out workers can consume in parallel and stay decoupled from the Post Service — if fan-out is slow, posting still succeeds; the queue absorbs the backpressure.

Fan-out-on-read (pull model) means: we don't push anything at write time. Instead, when a user opens their feed, the Feed Service queries the posts of everyone they follow, on demand, and merges/ranks them in real time. This makes writes trivially cheap — one write to the post store — but reads become expensive, since a user following 500 people needs 500 lookups merged and ranked on every feed open.

**4b. The celebrity/hot-key problem (Module 6: consistent hashing) — a hybrid approach.**
Neither pure strategy works well at our scale. The standard production answer, and what Twitter has publicly described, is a **hybrid**: use fan-out-on-write for the vast majority of users (average 500 followers — cheap to push), but for celebrity accounts above a follower threshold, skip the write-time fan-out entirely and merge their posts in at read time instead. When a normal user opens their feed, the Feed Service reads their precomputed timeline from cache (fast) and separately fetches recent posts from the small number of celebrities they follow (also fast, since a user follows only a handful of celebrities even if each celebrity has millions of followers), merging the two before ranking.

This is also a hot-key problem in the caching layer: a celebrity's post ID could get read by millions of clients within seconds. Consistent hashing across the Redis cluster distributes keys evenly across nodes and lets us add nodes without a full cache reshuffle, but for a genuinely hot key we still add read replicas or local/CDN-edge caching in front of that specific key to avoid overloading a single shard.

**4c. Caching precomputed timelines (Module 4).**
We use a cache-aside pattern: the precomputed timeline lives in Redis as a sorted set of post IDs per user, scored by rank/time. On a cache miss (e.g., a brand-new user, or an evicted key), the Feed Service falls back to rebuilding the timeline from the follow graph and post store, then repopulates the cache. TTLs and capped list lengths (e.g., only the latest ~800 entries) keep memory bounded given our ~24 TB estimate.

**4d. Sharding the post store (Module 3).**
Posts are sharded by author ID (or a hash of it) so that writing and reading a given user's own posts stays on one shard, and we use consistent hashing to minimize resharding pain as we add capacity — the same technique protecting our cache layer.

### Step 5: Bottlenecks & Trade-offs

- **Hot shard / hot key for celebrities**: even with hybrid fan-out, a celebrity's follow-graph entry and post-store row can become a hot key. Mitigate with read replicas, request coalescing, and edge caching.
- **Fan-out-on-write cost vs. fan-out-on-read latency**: push is write-heavy and storage-heavy; pull is read-heavy and latency-heavy. The hybrid trades a bit of read-path complexity for bounded worst-case write cost.
- **Ranking complexity vs. latency**: a more sophisticated ML ranking model improves relevance but adds compute time on the read path; we can precompute rough rankings during fan-out and do lightweight re-ranking at read time to balance this.
- **Eventual consistency**: followers may see a post a few seconds late, or in rare cases briefly out of order — an acceptable trade-off given our non-functional requirements.

### Recap

We clarified functional and non-functional requirements, sized the system at 300M DAU with millions of celebrity followers, designed a pipeline of Post Service → queue/pub-sub → Fan-out Service → cached timelines, and solved the celebrity hot-key problem with a hybrid fan-out strategy backed by consistent hashing and cache-aside precomputed timelines.

### What's Next

Try extending this design: how would you add real-time notifications, support post edits/deletes propagating through already-fanned-out caches, or move ranking to a streaming pipeline using the batch-vs-stream trade-offs from Module 5? The quiz file in this folder walks through several of these follow-ups with model answers.

## Key Takeaways

- Always separate functional requirements (post, follow, timeline, rank) from non-functional ones (latency, consistency, fan-out scale) before designing.
- Ground your architecture in real capacity numbers — DAU, posts/sec, follower distribution — since they determine which trade-offs matter.
- Fan-out-on-write optimizes reads at the cost of writes; fan-out-on-read optimizes writes at the cost of reads; a hybrid approach gets the best of both by routing celebrity accounts differently.
- Consistent hashing (Module 6) is the shared tool that protects both the sharded post store and the distributed cache from hot keys and uneven load.
- Cache-aside precomputed timelines (Module 4) plus pub-sub-driven fan-out (Module 5) plus sharded storage (Module 3) together form the backbone of a real-world feed system.
- Eventual consistency is an explicit, acceptable design choice here — don't over-engineer for strong consistency the requirements don't call for.
