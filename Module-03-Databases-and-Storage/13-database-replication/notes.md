# Notes: Database Replication

## Definitions

- **Replication** — the process of copying and maintaining data across multiple database nodes so more than one machine holds the same data, for availability, durability, and read scaling.
- **Leader / Master / Primary** — the node that accepts writes in a given topology.
- **Follower / Slave / Replica** — a node that holds a copy of the leader's data and typically serves reads.
- **Replication lag** — the delay between a write landing on the primary and that write becoming visible on a replica. Usually caused by asynchronous replication.
- **Failover** — the process of detecting a failed primary/master and promoting a replica/follower to take its place.
- **Read-your-writes problem** — a consistency bug where a user doesn't see their own recent write because it was read from a replica that hasn't caught up yet (a symptom of replication lag).
- **Write conflict** — in multi-leader systems, when two nodes accept concurrent writes to the same data and the values disagree once replicated to each other.
- **Last-write-wins (LWW)** — a conflict resolution strategy that keeps the write with the latest timestamp and discards the other.
- **Vector clock** — a data structure that tracks causal ordering of writes across nodes, used to detect true concurrency vs. one write causally following another.

## Master-Slave vs. Master-Master

| Dimension | Master-Slave (Leader-Follower) | Master-Master (Multi-Leader) |
|---|---|---|
| Write scaling | Limited — all writes go through one master | Better — writes accepted on multiple nodes |
| Read scaling | Strong — add followers to absorb read traffic | Strong — add leaders/followers similarly |
| Conflict handling | None needed — single writer, no conflicts | Required — needs LWW, vector clocks, or app-level merge |
| Complexity | Lower — simpler to reason about and operate | Higher — bidirectional replication + conflict resolution |
| Failover | Explicit promotion of a follower; brief write downtime possible | No single point of failure for writes; other masters keep serving |
| Typical use cases | Read-heavy apps (blogs, content sites, most CRUD apps) | Multi-region active-active apps, collaborative/global low-latency writes |

## Synchronous vs. Asynchronous Replication

| Dimension | Synchronous | Asynchronous |
|---|---|---|
| Write acknowledgment | Waits for replica(s) to confirm before ack to client | Acks client immediately, replicates in background |
| Durability | Strong — replica has data before write is confirmed | Weaker — window where only primary has the data |
| Write latency | Higher — pays network round trip per write | Lower — no round trip wait |
| Failure risk | Low data loss risk if primary dies | Possible data loss for un-replicated writes if primary dies |
| Common usage | Selectively, for critical writes / one replica | Default for most replicas in most production systems |

## Key Numbers / Rules of Thumb

- Asynchronous replication lag is often milliseconds under normal load but can grow to seconds (or more) under heavy write load or network issues.
- A common hybrid pattern: 1 synchronous replica (for durability) + N asynchronous replicas (for read scaling and geographic distribution).
- Read-heavy apps often run read:write ratios of 100:1 or higher — a strong signal that master-slave read replicas are the right first scaling lever.

## Quick Summary Bullets

- Replication = multiple copies of data across nodes, for availability + read scaling.
- Sync replication trades latency for durability; async trades a small durability risk for speed and creates replication lag.
- Master-Slave: one writer, many readers; simple; needs failover on master loss; watch for the read-your-writes problem.
- Master-Master: many writers; enables multi-region active-active; needs a conflict resolution strategy (LWW or vector clocks).
- Replication scales reads well (and writes, partially, in master-master); it does not solve total data size or ultimate write-throughput limits — that's what sharding is for.
