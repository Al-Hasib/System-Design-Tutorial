# Practice & Interview Questions

**1. Why does a local, in-process cache stop working well once you scale to multiple application servers?**
A local cache lives in a single server's memory, so different servers behind a load balancer each build their own separate, inconsistent copy of "the cache," causing redundant cache misses and duplicated memory usage across the fleet. A distributed cache solves this by giving every application server a single shared view of cached data over the network.

**2. List three key differences between Redis and Memcached.**
Redis supports rich data structures (lists, sets, sorted sets, hashes) while Memcached only supports simple key-value byte strings. Redis offers built-in persistence (RDB/AOF) and replication, while Memcached has neither. Redis also provides extra features like pub/sub and Lua scripting, whereas Memcached is a minimal, multi-threaded pure cache.

**3. When would you choose Memcached over Redis?**
Choose Memcached when your needs are strictly simple, high-throughput key-value caching with no requirement for persistence, replication, or complex data structures — its multi-threaded architecture can offer excellent raw throughput per node on straightforward get/set workloads with minimal operational complexity.

**4. What problem does consistent hashing solve when sharding a cache cluster?**
With naive modulo hashing (hash(key) % N), adding or removing a single node changes the assigned node for almost every key, causing a massive wave of cache misses. Consistent hashing maps keys and nodes onto a hash ring so that only a small fraction of keys need to move when the cluster's size changes, minimizing disruption.

**5. Explain how Redis Cluster distributes data across nodes.**
Redis Cluster divides the keyspace into a fixed 16,384 hash slots, and each node in the cluster owns a subset of those slots. When nodes are added or removed, only the affected slots (and their keys) are reassigned and migrated, rather than rehashing the entire keyspace.

**6. Why is Redis a common choice for storing user sessions in a horizontally scaled web application?**
Storing sessions in a shared Redis cache means any application server can retrieve a user's session data regardless of which server handles a given request, since the load balancer may route consecutive requests to different servers. This enables truly stateless application servers that can be scaled up or down freely without losing session continuity.

**7. What happens to data in Memcached if the server process restarts? How does this differ from Redis?**
Memcached has no persistence, so all cached data is lost on a restart — this is acceptable because Memcached is meant purely as a disposable cache, not a source of truth. Redis can optionally persist data to disk via RDB snapshots or an AOF log, allowing it to reload its dataset after a restart, though it's still generally best practice to treat even Redis as a cache backed by a real source of truth.

**8. Describe a real-world use case where Redis's data structures (beyond simple key-value) provide a clear advantage over Memcached.**
A leaderboard feature can use a Redis sorted set, where Redis natively maintains ranked ordering and supports efficient range queries (e.g., "top 10" or "user's rank") directly on the server. Implementing the same feature with Memcached's plain key-value model would require fetching and re-sorting data in the application, which is far less efficient.

**9. How does adding a distributed cache change the request latency profile compared to a local in-process cache, and why is it still usually worth it?**
A distributed cache adds a network round trip that a local in-process cache doesn't have, so it's slightly slower per request than an in-process lookup. It's still worth it because that added latency (sub-millisecond to a few milliseconds) is vastly smaller than the latency of hitting a database, while also solving the consistency and memory-duplication problems that local caches can't.

**10. In a flash-sale scenario with shared inventory counts across many application servers, why is a distributed cache with atomic operations (like Redis's INCR/DECR) important?**
Multiple application servers processing checkouts concurrently need to see and update the same inventory count in real time; atomic increment/decrement operations in a shared cache ensure concurrent decrements don't race or produce incorrect counts (which could cause overselling). A local cache per server would give each server a stale, independent view of inventory, making overselling far more likely.

**11. What is Redis pub/sub, and how might it be used alongside caching in a system?**
Redis pub/sub lets clients publish messages to named channels and other clients subscribe to receive them in real time. It's often used alongside caching for things like broadcasting cache-invalidation events to multiple application servers, or lightweight real-time notifications, without needing a separate messaging system for simple cases.

**12. If you needed both a fast cache and durability guarantees so cached computation results survive a restart, which technology from this video would you pick and why?**
Redis, because it supports persistence mechanisms (RDB snapshots and/or AOF logging) that allow it to reload its dataset after a restart, unlike Memcached which has no persistence at all and loses everything on restart.
