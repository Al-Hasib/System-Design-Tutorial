# Why This Topic Matters: Distributed Locking (Redlock, ZooKeeper & etcd)

> **In one sentence:** A mutex works because all the threads share memory and a clock; across machines there is neither, which is why distributed locks are far easier to get subtly, dangerously wrong than to get right.

## The World Before This Idea

You run a nightly billing job. It runs on a schedule inside your application. You scale to twelve instances. Now the billing job runs twelve times, and every customer is charged twelve times.

The obvious fix: "take a lock first." So each instance does `SET lock:billing my-id NX EX 60` in Redis, and only the winner runs.

That looks correct, and here is how it fails. Instance A takes the lock and begins work. A long garbage-collection pause freezes it for 90 seconds. The lock expires at 60. Instance B acquires it and starts billing. Instance A wakes up — with no idea any time has passed — and continues billing, believing it still holds the lock. Two instances, both convinced they are the exclusive owner, both writing.

Nothing in that sequence is a bug in your code. It is the default behavior of a lock with a timeout in a system where processes can pause and clocks can drift.

## The Problems It Solves

### 1. Duplicate execution of work that must run once
**What you see:** Scheduled jobs running N times on an N-instance fleet. Duplicate charges, duplicate emails, duplicate report generation.

**Why it happens:** Horizontal scaling replicates everything, including things that were implicitly singletons.

**How distributed locking solves it:** Exactly one instance acquires the lock and does the work; the rest skip it. The same mechanism provides leader election for singleton responsibilities — a scheduler, a compaction coordinator, a cluster manager.

### 2. Concurrent modification of a shared external resource
**What you see:** Two workers process the same queue item, or two services write the same file in object storage, and the result is interleaved garbage.

**Why it happens:** The resource has no transactional guarantee of its own. A database can protect a row; a file in S3 or a third-party API cannot.

**How locking solves it:** It serializes access to something that has no built-in serialization.

### 3. Locks that expire while the holder is still working
**What you see:** The failure described above — two holders at once, silently.

**Why it happens:** A lock must have a TTL, or a crashed holder would block the system forever. But a TTL means the lock can expire while a live-but-paused holder still believes it holds it. There is no way around this: **you cannot have both crash-safety and a guarantee that the holder knows it still holds the lock.**

**How fencing tokens solve it:** The lock service returns a monotonically increasing token with each grant. The holder passes that token to every write, and the storage system rejects any write carrying a token lower than the highest it has seen. Instance A wakes with token 33, instance B is already writing with token 34, and A's writes are rejected. This is the only construction that is actually safe, and it requires the *downstream system* to participate — which is why many real deployments simply cannot achieve true mutual exclusion and must rely on idempotency instead.

### 4. Locks that vanish on failover
**What you see:** With a Redis primary and replica, the primary grants a lock and crashes before replicating. The replica is promoted with no record of the lock, and grants it to someone else.

**Why it happens:** Asynchronous replication means the lock's existence was not durable at the moment it was granted.

**How consensus-based lock services solve it:** ZooKeeper and etcd commit lock state through a consensus protocol, so it is agreed by a majority before being acknowledged. A failover cannot lose it. They also offer ephemeral nodes and leases tied to a session — when the holder's session dies, the lock is released automatically, without relying on a fixed timeout guess. This is genuinely stronger than a single Redis key, and it is why critical coordination lives in etcd rather than in a cache.

## The Price You Pay

- **Locks serialize, and serialization is the opposite of scaling.** Every lock is a bottleneck by construction. A coarse lock around a hot path can erase the benefit of your entire fleet.
- **The lock service is a hard dependency.** If it is down, work stops. Failing open means duplicate execution; failing closed means an outage. You must choose deliberately.
- **Redlock is disputed.** The multi-node Redis locking algorithm has been the subject of a well-known technical debate about whether it provides real safety under clock drift and process pauses. The practical takeaway is not which side is right: it is that **if correctness genuinely matters, use a consensus-backed service and fencing tokens; if you are only optimizing away duplicate work, a simple Redis lock is fine.** Being clear about which situation you are in is the actual skill.
- **Latency on every acquisition.** A consensus round trip per lock is not cheap, which is another reason to lock narrowly and briefly.
- **Deadlocks and lock leaks.** A holder that crashes without releasing, or two holders taking locks in different orders, cause the same problems as in a single process — with slower detection.

**And the strongest move is often to avoid the lock entirely.** An idempotent operation needs no mutual exclusion. A conditional atomic update in the database (`UPDATE ... WHERE status = 'pending'`) achieves exclusion using the guarantee the database already provides. Partitioning work by key so only one worker can ever own a given key removes contention by construction. Reach for a distributed lock after these, not before.

## When You Need It — and When You Don't

| You need a distributed lock when | Use something else when |
|---|---|
| Exactly one instance may perform an action | The operation can be made idempotent |
| You need leader election for a singleton role | The database can enforce it with a conditional update |
| A shared external resource has no transaction support | Work can be partitioned so owners never overlap |
| Correctness depends on true mutual exclusion | You only want to reduce duplicate work, not prevent it |

## Why This Shows Up in Interviews

Any design with scheduled jobs, singleton workers, or contention over a shared resource can surface this. The interviewer is usually testing whether you know the hard part — the expiry problem. Saying "I would take a Redis lock" is the entry-level answer. Saying "I would take a lock with a TTL, but a lock alone is not sufficient, because a GC pause can let the TTL expire while the holder still thinks it owns it — so either I use fencing tokens with a consensus-backed service like etcd, or, preferably, I make the operation idempotent so a duplicate is harmless" is a genuinely senior answer.

## How It Connects

Correct distributed locking is built on **consensus** (topic 27), which is what makes etcd and ZooKeeper trustworthy for it. Simpler locks are built on **Redis** (topic 19). **Idempotency** (topic 29) is both the alternative to locking and the safety net when locking fails. It is the distributed analogue of the concurrency control in **topic 37**, becomes necessary because of **horizontal scaling** (topic 4), and is used constantly inside **Kubernetes** (topic 44) for leader election among controllers.

**Next:** [Logical Clocks & Time in Distributed Systems](../41-logical-clocks-and-time-in-distributed-systems/why.md) — why the clock you just relied on cannot be trusted.
