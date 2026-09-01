# Distributed Locking: Redlock, ZooKeeper & etcd

**Difficulty:** Advanced
**Estimated length:** 14-18 min
**Prerequisites:** [27 - Consensus Algorithms: Paxos and Raft](../../Module-06-Distributed-Systems-Concepts/27-consensus-algorithms-paxos-and-raft/README.md), [19 - Distributed Caching with Redis & Memcached](../../Module-04-Caching-and-Content-Delivery/19-distributed-caching-redis-and-memcached/README.md)

## Learning Objectives

- Explain why an ordinary in-process lock (a mutex) can't coordinate access across multiple machines.
- Describe the Redlock algorithm and the specific race condition it's designed to prevent.
- Explain why ZooKeeper and etcd, built on consensus, offer stronger correctness guarantees for distributed locks than a single Redis instance.
- Understand the fencing token pattern and why a lock alone isn't enough to guarantee mutual exclusion in a distributed system.
- Choose an appropriate distributed locking approach for a given scenario's correctness requirements.

## Script

### Hook / Intro

An ordinary lock — a mutex in your programming language of choice — works because every thread that might contend for it lives inside the same process, sharing the same memory. The moment you have multiple independent servers, each running their own copy of your application, that entire mechanism stops existing. And yet the underlying problem doesn't go away: you often still need exactly one of many processes to do something — send a scheduled email exactly once, process a specific job exactly once, hold exclusive access to a resource while it's being modified. This is distributed locking, and today we cover the two dominant approaches — Redis-based (Redlock) and consensus-based (ZooKeeper/etcd) — along with a subtlety that catches almost everyone the first time: a lock isn't automatically safe just because you "have" it.

### Why This Is Hard

In a single process, a mutex guarantees mutual exclusion because the operating system enforces it directly on shared memory — there's no ambiguity about who holds the lock right now. In a distributed system, "the lock" has to live somewhere external that every process can see — commonly a shared data store like Redis, or a dedicated coordination service like ZooKeeper or etcd — and every interaction with it happens over an unreliable network, with its own latency, possible partitions, and possible node failures. The core question a distributed lock has to answer isn't just "who has the lock" — it's "how do we handle a lock holder we can no longer reach, without either deadlocking forever or letting two holders both believe they have exclusive access at once."

### Redlock: A Redis-Based Approach

The most common lightweight approach uses Redis: acquiring a lock is just setting a key (with a unique value identifying the holder) that automatically expires after a timeout — `SET lock_key unique_value NX PX 30000` sets it only if it doesn't already exist, with a 30-second expiry. If the process crashes or loses connectivity, the lock isn't held forever; it simply expires and another process can acquire it. Releasing the lock deletes the key, but only if the value still matches — this prevents a process from accidentally releasing a lock it no longer actually holds (say, after its own lock already expired and someone else acquired it).

A single Redis instance is a single point of failure, though — if that Redis node goes down, every lock is unavailable, or worse, could be lost mid-hold. **Redlock**, proposed by Redis's creator, addresses this by running the same lock acquisition against **five independent Redis instances** and requiring a majority (three of five) to succeed within a tight time budget before considering the lock acquired. The idea is that even if a couple of Redis nodes fail or are slow, the lock can still be safely acquired and released as long as a majority agree — borrowing the "majority quorum" idea from consensus algorithms (recall Raft from Module 6) without requiring a full consensus protocol.

Redlock is genuinely controversial in distributed systems circles — a widely-cited critique from Martin Kleppmann (author of *Designing Data-Intensive Applications*) argues Redlock's timing assumptions (that clocks across five nodes stay reasonably synchronized, that a process's own clock doesn't experience a long pause, e.g. from garbage collection, at exactly the wrong moment) can be violated in ways that let two clients believe they simultaneously hold the same lock. This doesn't mean Redlock is useless — for many practical use cases (avoiding duplicate work, not perfect correctness under adversarial conditions) it's a reasonable, fast, low-overhead tool. But it's worth knowing this is not a mathematically airtight mutual-exclusion guarantee, which matters if your use case genuinely can't tolerate the rare case of two holders.

### ZooKeeper / etcd: Consensus-Based Locking

For correctness guarantees that don't rely on synchronized clocks, ZooKeeper and etcd take a fundamentally different approach: they're built directly on a consensus protocol (ZooKeeper uses ZAB, etcd uses Raft — both covered conceptually in Module 6), meaning a cluster of these nodes maintains one strongly-consistent, agreed-upon view of state, tolerating the failure of a minority of nodes without ever disagreeing about who holds a lock. A typical ZooKeeper-based lock works by having each contender create a sequential, ephemeral node under a shared path; the contender with the lowest sequence number holds the lock, and everyone else watches the node just ahead of them in line, getting notified when it's their turn. "Ephemeral" means the node is automatically removed if that client's session dies (detected via heartbeats to the ZooKeeper cluster) — so a crashed lock holder doesn't block everyone else forever, without relying on wall-clock timeout guessing the way Redlock does. This buys stronger correctness at the cost of running (and operating) a dedicated, non-trivial coordination cluster rather than reusing a cache you might already have.

### The Fencing Token Problem

Here's the subtlety that catches people even with a "correct" lock: acquiring a lock doesn't guarantee that the holder acts on it in a timely, uninterrupted way. Imagine a client acquires a lock, then experiences a long pause (garbage collection, a slow network, being descheduled by the OS) *after* acquiring the lock but *before* actually using it to write to some shared storage. Meanwhile, the lock could have expired and been acquired by a second client, which does its work and releases it. When the first client's pause ends, it may resume and write to storage — completely unaware its lock is no longer valid, having no idea a second client even existed. The lock's existence was checked correctly, but its assumption (that holding the lock corresponds to exclusive time to act) turned out to be false. The standard fix is a **fencing token**: every time a lock is granted, it comes with a strictly increasing number. When the client writes to shared storage, it includes this token, and the storage system itself rejects any write carrying an older token than one it's already seen — so even if a delayed client tries to act after its lock has effectively expired, its stale token is rejected at the point of actual effect, not just at the point of lock acquisition.

### Real-World Example

Consider a distributed cron system where a scheduled job (say, "send the daily digest email") runs on one of several worker instances, and it absolutely must not run twice. Using Redlock against a Redis cluster your team already operates is fast to set up and good enough if an occasional duplicate email would be an annoyance rather than a real incident. If instead this were "charge a customer's card for a subscription renewal" — where a duplicate execution is a genuine financial and trust problem — you'd want the stronger guarantees of ZooKeeper or etcd's consensus-backed locking, and you'd likely add a fencing token on the actual charge-processing step so that even a delayed, "zombie" holder of an expired lock can't cause a real double-charge.

### Recap

Distributed locking exists because ordinary in-process mutexes can't coordinate across independent machines — the lock has to live in a shared, network-accessible place, and has to handle a holder that becomes unreachable without deadlocking or double-granting. Redlock uses a majority of independent Redis instances for a fast, practical lock, with known, debated timing-assumption weaknesses. ZooKeeper and etcd use consensus protocols to give stronger correctness guarantees at the cost of running dedicated coordination infrastructure. And regardless of which lock you use, a fencing token — a strictly increasing number checked at the point of actually writing to shared storage — is what closes the gap between "held the lock" and "the action the lock was protecting actually happened safely."

### What's Next

We've covered how distributed processes coordinate exclusive access to a resource. Next video tackles a related but distinct problem: how a distributed system agrees on the *order* events happened in, when there's no single shared clock every machine can trust.

## Key Takeaways

- Ordinary mutexes can't coordinate across machines because there's no shared memory — a distributed lock must live in an external, network-accessible store and handle unreachable holders without deadlocking or double-granting.
- Redlock acquires a lock against a majority of independent Redis instances with an expiry — fast and practical, but has known, debated timing-assumption weaknesses (clock skew, process pauses).
- ZooKeeper and etcd build locks on top of a consensus protocol (ZAB/Raft), giving stronger correctness guarantees at the cost of operating dedicated coordination infrastructure.
- Holding a lock doesn't guarantee safe, uninterrupted action — a process can pause after acquiring a lock and act after it's effectively expired.
- Fencing tokens (a strictly increasing number checked at the point of actually writing to shared storage) close this gap, rejecting stale actions even from a "zombie" lock holder.
