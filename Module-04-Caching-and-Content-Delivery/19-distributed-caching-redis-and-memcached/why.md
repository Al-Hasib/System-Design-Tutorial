# Why This Topic Matters: Distributed Caching with Redis & Memcached

> **In one sentence:** The moment you run more than one application server, an in-process cache stops being a cache and becomes N inconsistent caches — a shared cache tier is what fixes that, and it turns out to solve half a dozen other problems too.

## The World Before This Idea

You cache in local process memory. It's fast, it's free, and it works beautifully — on one server.

Add a second server and the problems begin. Each has its own copy, so a user can see one value on one request and a different value on the next, at random. Hit rates collapse: with ten servers, a cached item is useful to only 1/10th of traffic, so you're doing ten times the work to warm ten copies. When a value changes, you have to invalidate it on every node — and there's no reliable mechanism to do that. Deploys wipe every cache simultaneously, dropping full traffic onto the database at the worst possible moment.

Meanwhile some things simply *cannot* live in local memory: a session must be visible to whichever server the load balancer picks, a rate-limit counter must be global to mean anything, and a lock is worthless if it's per-process.

## The Problems It Solves

### 1. Cache inconsistency across a fleet
**What you see:** Users see values flip between old and new on refresh. Invalidation "works" but only sometimes.

**Why it happens:** N independent caches with no coordination.

**How a shared cache solves it:** One logical cache, one value per key, visible to every server. Invalidate once and it's invalidated for everyone. This is the primary reason to move off local caching.

### 2. State that must be shared to be correct
**What you see:** Sessions break when a user's requests land on different servers. Rate limits allow 10x the intended traffic because each server counts independently. A scheduled job runs on all twelve nodes at once.

**Why it happens:** All three require *globally* visible state. Per-process state gives per-process answers.

**How Redis solves it:** A fast shared store with atomic operations. `INCR` for counters, `SETNX` with expiry for locks, plain keys with TTL for sessions. Redis's atomicity is what makes these correct under concurrency — the operations complete without interleaving, which is exactly what a rate limiter or lock needs.

### 3. Cold caches after every deploy
**What you see:** Every deploy causes a latency spike and a database load spike.

**Why it happens:** In-process caches die with the process.

**How an external cache solves it:** The cache tier has its own lifecycle. Application servers restart, deploy, and autoscale freely; the cache stays warm throughout. This decoupling is worth more than it sounds — it removes a recurring, self-inflicted load spike.

### 4. Data structures the database shouldn't be doing
**What you see:** A leaderboard recomputed with an expensive `ORDER BY ... LIMIT` on every page load. A "recently viewed" list maintained with delete-and-insert transactions.

**Why it happens:** Treating a relational database as a general-purpose data structure engine.

**How Redis solves it:** Sorted sets give O(log n) leaderboards natively. Lists, sets, hashes, HyperLogLog, and streams each replace a chunk of application logic with a single fast operation. This is where Redis stops being "a cache" and becomes a data structure server — and it's the main reason it's chosen over Memcached, which is deliberately simpler (strings only, multithreaded, excellent at exactly one job).

## The Price You Pay

- **A network hop.** Local memory is nanoseconds; Redis is a sub-millisecond network round trip. Still enormously faster than a database, but no longer free — which means chatty code that makes 50 cache calls per request has just moved its bottleneck rather than removing it (use pipelining or multi-get).
- **A new critical dependency.** If sessions live in Redis and Redis goes down, nobody can log in. It needs replication, failover, and monitoring like any datastore.
- **Memory is finite and eviction is real.** When memory fills, keys get evicted by policy. If you're using it as a durable store, that's data loss — and Redis's persistence options (RDB, AOF) make durability *possible* but not free, and not equivalent to a database.
- **Distribution problems.** Sharding a cache across nodes raises the "which node holds this key" question, and naive modulo hashing reshuffles everything when the cluster changes — which is exactly why consistent hashing exists.
- **It's another system to operate.** Version upgrades, memory tuning, cluster topology, and failover behavior are all real work.

## When You Need It — and When You Don't

| Use a shared cache when | Local cache is fine when |
|---|---|
| You run multiple application instances | Single instance, or data is immutable |
| State must be globally consistent (sessions, counters, locks) | The value is server-local by nature (compiled templates, config) |
| You want the cache to survive deploys | Sub-microsecond access genuinely matters |
| You need rich data structures (Redis) | — |
| Hit rate matters and you can't afford N copies | — |

Many mature systems run both: a tiny local cache in front of the shared cache for the hottest keys, accepting brief inconsistency for the very top of the distribution.

## Why This Shows Up in Interviews

Redis appears in most designs, so naming it isn't the signal. The signals are: choosing Redis over Memcached for a specific reason (data structures, persistence, Pub/Sub) rather than by habit; using the right primitive for the job (sorted set for a leaderboard, `INCR` for rate limiting, `SETNX` with TTL for a lock); and answering "what if Redis goes down?" with something better than silence. Rate limiter and leaderboard questions in particular are really Redis data-structure questions in disguise.

## How It Connects

This is **caching strategy** (topic 17) made concrete. It's the shared state layer that makes **horizontally scaled** stateless servers possible. It's the standard implementation substrate for **rate limiting** (topic 25), **distributed locking** (topic 40), **Pub/Sub** fan-out behind **WebSockets**, and several **probabilistic data structures** (topic 42). Scaling the cache tier itself is what motivates **consistent hashing** (topic 24).

**Next:** [Message Queues: Kafka vs RabbitMQ](../../Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/why.md) — decoupling work in time, not just caching it.
