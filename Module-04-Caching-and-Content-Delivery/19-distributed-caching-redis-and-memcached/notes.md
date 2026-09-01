# Study Notes: Distributed Caching with Redis & Memcached

## Definitions

- **Local (in-process) cache**: A cache stored in a single application instance's memory; fast, but inconsistent and duplicated across multiple instances.
- **Distributed cache**: A shared caching layer, external to application processes, accessed over the network by all application instances, providing a single consistent view of cached data.
- **Sharding**: Splitting a dataset across multiple cache nodes, typically by hashing each key to determine its assigned node.
- **Consistent hashing**: A hashing scheme that minimizes key redistribution when nodes are added/removed from a cluster (covered in depth in Module 6).
- **RDB (Redis Database file)**: Point-in-time snapshot persistence mechanism in Redis.
- **AOF (Append-Only File)**: Write-ahead-log-style persistence mechanism in Redis that logs every write operation.
- **Pub/Sub**: A messaging pattern where publishers send messages to channels and subscribers receive them; supported natively by Redis.

## Redis vs Memcached

| Feature | Redis | Memcached |
|---|---|---|
| Data model | Rich: strings, lists, sets, sorted sets, hashes, streams, geospatial | Simple key-value (byte strings only) |
| Persistence | Yes — RDB snapshots and/or AOF log | No — data lost on restart |
| Replication | Yes — primary/replica replication built in | No native replication |
| Clustering / Sharding | Redis Cluster (16,384 hash slots across nodes) | Client-side sharding (consistent hashing typically implemented in the client) |
| Threading model | Primarily single-threaded event loop for command execution (some I/O threading in newer versions) | Multi-threaded |
| Extra features | Pub/Sub, transactions (MULTI/EXEC), Lua scripting, atomic ops on complex types | None beyond basic get/set/incr |
| Eviction policies | Multiple configurable policies (LRU, LFU, random, TTL-based, etc.) | LRU |
| Typical use cases | Caching + session store + leaderboards + rate limiting + pub/sub + lightweight queues | Simple, high-throughput key-value caching |
| Maturity/complexity | More features, more configuration surface | Minimal, easy to operate |

## Local Cache vs Distributed Cache

| Aspect | Local (In-Process) Cache | Distributed Cache |
|---|---|---|
| Speed | Fastest (no network hop) | Slower than local, still much faster than DB |
| Consistency across servers | Inconsistent — each instance has its own copy | Consistent — single shared view |
| Memory efficiency | Duplicated per instance | Stored once, shared |
| Scales with app servers? | Cache "resets" effectively as you add more instances | Cache scales independently of app server count |

## Sharding / Data Distribution

| Approach | Description | Weakness |
|---|---|---|
| Modulo hashing (hash(key) % N) | Simple, deterministic | Adding/removing a node reshuffles almost all keys |
| Consistent hashing | Keys and nodes mapped onto a hash ring; only a fraction of keys move when nodes change | More complex to implement correctly |
| Redis Cluster hash slots | Fixed 16,384 slots distributed across nodes; slots reassigned (not full rehash) when nodes change | Redis-specific mechanism |

## Key Numbers / Rules of Thumb

- Distributed cache access typically adds sub-millisecond to low-single-digit-millisecond network latency versus a local in-process cache, but remains far faster than typical database queries (often 10-100x faster).
- Redis Cluster uses exactly 16,384 hash slots, regardless of cluster size.
- Memcached's multi-threaded architecture can yield higher raw throughput per node on simple get/set workloads on machines with many CPU cores.

## Summary

- Local caches don't work once you scale beyond one application server — they cause inconsistency and duplicated memory.
- A distributed cache is a shared, network-accessible caching layer used by all application instances.
- Memcached: simple, multi-threaded, pure key-value, no persistence/replication — great for straightforward high-throughput caching.
- Redis: rich data structures, persistence, replication, clustering, pub/sub — the more versatile, commonly-default choice.
- Data is spread across cache clusters via sharding, ideally using consistent hashing to minimize disruption when scaling the cluster.
