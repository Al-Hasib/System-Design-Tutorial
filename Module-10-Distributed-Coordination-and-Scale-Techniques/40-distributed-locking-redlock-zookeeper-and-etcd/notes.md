# Study Notes: Distributed Locking

## Definitions

- **Distributed lock:** A mechanism ensuring only one process among many, on different machines, can hold exclusive access to a resource at a time.
- **Redlock:** A Redis-based distributed locking algorithm requiring a majority (e.g., 3 of 5) of independent Redis instances to agree a lock is acquired, within a time budget.
- **Ephemeral node (ZooKeeper):** A node that automatically disappears if the client session that created it dies — used to detect a crashed lock holder without relying on timeouts alone.
- **Fencing token:** A strictly increasing number issued with each lock grant; the protected resource rejects any action carrying an older token than one already seen, preventing a delayed/stale holder from causing damage.

## Approaches Compared

| Approach | Foundation | Correctness guarantee | Operational cost | Best fit |
|---|---|---|---|---|
| Redlock | Majority of independent Redis instances + expiry | Practical, but debated timing assumptions (clock sync, process pauses) | Low — reuse Redis you may already run | Non-critical deduplication (e.g., avoid duplicate low-stakes jobs) |
| ZooKeeper / etcd | Consensus protocol (ZAB / Raft) | Strong — tolerates minority node failure without disagreement | Higher — dedicated coordination cluster to run/operate | High-stakes exclusive access (e.g., leader election, critical single-execution jobs) |

## Why a Lock Alone Isn't Enough: The Fencing Token Problem

1. Client A acquires a lock.
2. Client A pauses (GC, network delay, OS scheduling) *after* acquiring but *before* acting.
3. Lock expires; Client B acquires it, does its work, releases it.
4. Client A resumes, unaware its lock is stale, and acts anyway — potential conflict/corruption.
5. **Fix:** Each lock grant includes a strictly increasing fencing token. The protected resource (e.g., storage system) rejects any write with a token older than the newest one it has already accepted — so Client A's stale action is rejected at the point of effect, even though its lock-acquisition check originally succeeded.

## Key Numbers / Facts

- Redlock's reference implementation typically uses 5 independent Redis instances and requires acquisition on a majority (3) within a bounded time window.
- ZooKeeper uses the ZAB (ZooKeeper Atomic Broadcast) consensus protocol; etcd uses Raft — both covered conceptually in Module 6's consensus video.
- Martin Kleppmann's widely-cited critique of Redlock ("How to do distributed locking," 2016) argues its safety depends on assumptions (bounded clock drift, no long process pauses) that aren't guaranteed in real systems.

## Summary

- Distributed locks solve the same mutual-exclusion problem as an in-process mutex, but must live in a shared, network-accessible store and handle unreachable holders without deadlocking or double-granting.
- Redlock is fast and practical but has known, debated weaknesses; ZooKeeper/etcd give stronger, consensus-backed correctness at higher operational cost.
- A lock by itself doesn't guarantee the protected action happens safely — fencing tokens close the gap between "held the lock" and "the action actually happened without conflict."
