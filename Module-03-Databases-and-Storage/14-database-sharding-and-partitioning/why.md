# Why This Topic Matters: Database Sharding & Partitioning

> **In one sentence:** Replicas let you scale reads without limit, but every write still lands on one node — sharding is the only way past that wall, and it's the most expensive, least reversible thing you can do to a database.

## The World Before This Idea

You've indexed everything, added read replicas, cached aggressively. Reads are fine. But you're writing 80,000 rows per second and the primary is saturated. There is no bigger machine, and you can't add replicas to fix writes — replicas *receive* the same writes, so they don't reduce the load at all.

Meanwhile your table has grown past 8 TB. Backups take a day. Index rebuilds take a weekend. A schema migration is a multi-week project. The dataset itself, not just the traffic, has outgrown a single machine.

## The Problems It Solves

### 1. A write ceiling replicas can't raise
**What you see:** The primary's disk and CPU are pinned. Adding replicas does nothing.

**Why it happens:** Every write must be applied on the primary, and replicas apply the same write stream. Replication scales reads, not writes — this is the single most important thing to understand about it.

**How sharding solves it:** Split the data by key across N independent primaries. Each shard receives roughly 1/N of the writes and owns its own disk, CPU, and memory. Write throughput now grows with the number of shards.

### 2. A dataset too large to operate on
**What you see:** Backups, restores, migrations, and index builds all take so long they're effectively impossible to do safely.

**Why it happens:** Operational time scales with data volume on a single node.

**How sharding solves it:** Each shard is a manageable size. You back up, migrate, and rebuild shard by shard, in parallel, and a problem with one shard affects 1/N of users instead of everyone. Blast radius shrinks along with data size.

### 3. Hot data and cold data competing
**What you see:** Queries against this month's data are slow because they compete with five years of archived rows in the same table and the same buffer pool.

**Why it happens:** All data is equally present regardless of how often it's touched.

**How partitioning solves it:** Range-partitioning by time keeps recent data in small, hot partitions that stay in memory, while old partitions sit cold. Dropping last year's data becomes an instant partition drop instead of a `DELETE` that runs for six hours and bloats the table.

### 4. Choosing a shard key that ruins everything
**What you see:** You sharded, and one shard is at 95% CPU while the other nine idle. Or every query now has to ask all ten shards and wait for the slowest.

**Why it happens:** The shard key determines everything. Shard by country and one country dominates. Shard by timestamp and *all* current writes land on the newest shard — a hotspot by construction. Shard by a key your queries don't filter on and every query becomes a scatter-gather.

**How this topic solves it:** It makes the shard key the central decision, chosen against three criteria: even distribution, alignment with your dominant query pattern, and stability over time. Getting this right is most of the work; getting it wrong means resharding, which is the painful part.

## The Price You Pay

Sharding is a genuine architectural downgrade in everything except scale, and that's the honest framing:

- **Cross-shard queries are slow or impossible.** A query that doesn't include the shard key must hit every shard and merge results — as slow as the slowest shard, and much harder to paginate or sort.
- **Cross-shard joins effectively don't exist.** You denormalize, duplicate data, or join in the application.
- **Cross-shard transactions effectively don't exist.** ACID is per-shard. Anything spanning shards needs two-phase commit (slow, fragile) or sagas (eventually consistent, complex).
- **Resharding is brutal.** Changing the shard key or adding shards means moving enormous amounts of data while live. Consistent hashing reduces the pain but doesn't remove it.
- **Operational multiplication.** Ten shards, each with replicas, means dozens of nodes to monitor, back up, patch, and fail over. Every operation is now a fleet operation.
- **Hotspots persist.** Even with a decent key, one celebrity user or one viral item can overwhelm a single shard.

**Because of all this, sharding should be last.** Index properly, cache, add replicas, archive old data, upgrade hardware, and split by service first. Teams routinely shard years before they need to and pay the complexity every day for a scale that never arrives.

## When You Need It — and When You Don't

| Shard when | Do something else when |
|---|---|
| Write throughput exceeds one node and you've exhausted tuning | Reads are the bottleneck → replicas and caching |
| The dataset is too large to back up or migrate | Queries are slow → indexes |
| You have a natural, high-cardinality partition key (user ID, tenant ID) | Growth is mostly old data → archive or time-partition |
| You need per-tenant or per-region data isolation | You're still on modest hardware — scale up first |

## Why This Shows Up in Interviews

Any design at internet scale reaches a point where a single database can't hold the data — news feeds, chat, ride-sharing, file storage all do. Interviewers want to hear: what's the shard key, *why that key*, what queries become expensive because of it, how you handle hotspots, and what happens when you add a shard. Naming a shard key and then immediately identifying the queries it makes awkward is a strong signal. Saying "we'll shard by user ID" with no discussion of consequences is not.

## How It Connects

Sharding is the write-scaling counterpart to **replication**'s read scaling, and production systems combine them (each shard is itself replicated). **Consistent hashing** is the standard technique for mapping keys to shards so that adding a node moves minimal data. Losing cross-shard transactions is precisely why **distributed transactions (2PC and Saga)** exist. **CAP theorem** applies because a sharded cluster is a distributed system with partitions. And **denormalization** becomes near-mandatory once joins are off the table.

**Next:** [CAP Theorem & PACELC](../15-cap-theorem-and-pacelc/why.md) — the theory that explains the trade-offs you just ran into.
