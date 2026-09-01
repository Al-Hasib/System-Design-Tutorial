# Distributed Caching with Redis & Memcached

**Difficulty:** Intermediate/Advanced
**Estimated Length:** 13-16 min
**Prerequisites:** [17 - Caching Strategies & Cache Invalidation](../17-caching-strategies-and-cache-invalidation/README.md), [13 - Database Replication](../../Module-03-Databases-and-Storage/13-database-replication/README.md)

## Learning Objectives

- Explain why a shared, distributed cache is needed once you scale to multiple application servers
- Compare Redis and Memcached across data structures, persistence, replication, and clustering
- Understand how data is distributed across a cache cluster (sharding/consistent hashing at a high level)
- Describe Redis-specific features: data structures, persistence (RDB/AOF), pub/sub, and replication
- Identify appropriate use cases for a dedicated distributed cache versus a per-instance local cache

## Script

### Hook / Intro

So far we've talked about caching data near your application and caching content near your users with a CDN. But here's a question: what happens when your application isn't just one server anymore, but a hundred servers behind a load balancer? If each of those hundred servers keeps its own local, in-memory cache, you've got a hundred separate, inconsistent copies of "the cache" — and a hundred times the cold-start misses. The solution is a **distributed cache**: a shared caching layer that every application server talks to over the network. In this video, we'll dig into the two most popular technologies for this — Redis and Memcached — and figure out when to reach for each.

### Why You Need a Distributed, Shared Cache

Let's set the scene. Imagine a "local cache" — literally a hash map living in the memory of each application server's process. It's blazing fast, since there's no network hop at all. But once you have multiple servers, this falls apart in two ways. First, consistency: if a user's request goes to server A, gets cached there, and their next request lands on server B due to load balancing, server B has no idea that data exists — it's a cache miss all over again. Second, memory efficiency: you're storing the same data redundantly across every server's memory instead of once, centrally.

A distributed cache solves both problems by moving the cache out of individual application processes and into its own dedicated service — a cluster of caching servers that all application servers talk to over the network. Now every application server sees the same cached data, and you only store each piece of data once, in the cache cluster, no matter how many application servers you scale out to. The trade-off is that you've added a network hop, so it's a bit slower per request than a local, in-process cache, but it's still vastly faster than hitting a database, and it solves the consistency and duplication problems local caches can't.

### Memcached: Simple, Fast, and Focused

**Memcached** is one of the oldest and simplest distributed caching systems, and that simplicity is its main selling point. It's a pure key-value store: you put a key and a byte-string value in, and you get that value back out by key, until it expires or gets evicted. That's essentially it. Memcached is multi-threaded, which lets it take excellent advantage of multi-core machines for very high throughput on simple operations, and it uses a straightforward LRU eviction policy when it runs low on memory. It has no built-in persistence — if a Memcached server restarts, everything in it is gone — and no built-in replication; it's meant to be a pure, disposable, high-speed cache layer, not a source of truth for anything.

### Redis: A Data Structure Server

**Redis** started from the same idea — an in-memory key-value cache — but grew into something more powerful: an in-memory data structure server. Beyond simple strings, Redis natively supports lists, sets, sorted sets, hashes, streams, and even geospatial data types, with efficient operations built directly into the server. This means you can do things like maintain a leaderboard using a sorted set, where Redis handles the ranking for you, or use a list as a lightweight queue — use cases that would require a lot of custom application logic with Memcached's plain key-value model.

Redis also supports **persistence**: it can periodically snapshot its dataset to disk (called RDB snapshots) or maintain an append-only log of every write (AOF, Append-Only File) so it can recover its data after a restart — something Memcached simply cannot do. Redis supports **replication**, running as a primary with one or more read replicas, and **Redis Cluster** for horizontal scaling and sharding across multiple nodes with built-in high availability. It also has extra features like **pub/sub messaging** and support for atomic transactions on certain operations.

### Redis vs Memcached: Choosing Between Them

So which do you pick? If your need is purely "store small values, retrieve them fast, high throughput, nothing fancy," Memcached is lean, mature, and excellent at exactly that job, particularly for multi-threaded workloads on large multi-core machines. If you need richer data structures, persistence, replication for high availability, pub/sub, or you want your caching layer to double as a lightweight message broker or session store, Redis is the more versatile choice — and in practice, Redis has become the more commonly reached-for default in modern system design, precisely because of that versatility, even in many simple caching scenarios.

### Distributing Data Across a Cache Cluster

Whichever technology you choose, once you have more data than fits on a single cache server — or need more throughput than one server can handle — you need to spread data across multiple cache nodes. This is where **sharding** comes in: each key is deterministically assigned to one specific node, typically using a hashing scheme. A naive approach — hash the key modulo the number of nodes — has a big weakness: if you add or remove a node, nearly every key's assigned node changes, causing a massive wave of cache misses. That's why production systems typically use **consistent hashing**, a technique that minimizes how many keys need to move when the cluster changes size, keeping the disruption localized to a small fraction of the cache instead of nearly all of it. We'll cover consistent hashing in much more depth in Module 6, but it's worth knowing now that this is the standard approach behind Memcached client-side sharding and Redis Cluster's internal data distribution (Redis Cluster technically uses a fixed set of 16,384 hash slots distributed across nodes, which achieves a similar rebalancing benefit).

### Real-World Example

Think about an e-commerce site during a flash sale. Product inventory counts are read constantly by every checkout attempt across dozens of application servers. Without a shared cache, each server might have a stale, locally-cached inventory count, leading to overselling. With a distributed Redis cluster in front of the database, every application server checks and decrements the same shared inventory counter, using Redis's atomic increment/decrement operations, keeping the numbers consistent across the whole fleet in real time, all while sparing the database from being hammered by every single page view and checkout click.

Another classic example: **session storage**. In a load-balanced web application, a user's session data needs to be available no matter which server handles their next request. Storing sessions in a distributed Redis cache — instead of in each server's local memory — means any server can retrieve a logged-in user's session instantly, enabling truly stateless, horizontally scalable application servers.

### Recap

Let's recap. Once you scale beyond a single application server, local in-process caches break down due to inconsistency and duplication, so you move to a distributed cache that all servers share over the network. Memcached is a simple, multi-threaded, pure key-value cache — great for straightforward, high-throughput caching with no persistence needs. Redis is a richer in-memory data structure server, supporting complex types, persistence, replication, and clustering, making it the more flexible default for most modern systems. And when your data outgrows a single cache node, sharding — ideally via consistent hashing — spreads it across a cluster while minimizing disruption when nodes are added or removed.

### What's Next

We've now built up a full picture of caching — application-level strategies, CDNs at the network edge, and distributed caches like Redis and Memcached tying it all together across a fleet of servers. That wraps up Module 4. In Module 5, we shift into messaging and asynchronous systems, starting with message queues and comparing two of the biggest names in that space: Kafka and RabbitMQ.

## Key Takeaways

- A distributed cache is necessary once you have multiple application servers, because local in-process caches cause inconsistency and duplicated memory usage across instances.
- Memcached is a simple, multi-threaded, pure key-value store with no built-in persistence or replication — optimized for raw speed and simplicity.
- Redis is an in-memory data structure server supporting rich types (lists, sets, sorted sets, hashes), persistence (RDB/AOF), replication, clustering, and pub/sub — making it more versatile.
- Redis Cluster and consistent hashing techniques allow cache data to be sharded across multiple nodes while minimizing key movement when nodes are added or removed.
- Common distributed cache use cases include database query caching, session storage, rate limiting counters, leaderboards, and real-time counters (e.g., inventory).
- Choose Memcached for simple, maximal-throughput key-value caching; choose Redis when you need richer data structures, durability, or high-availability replication.
