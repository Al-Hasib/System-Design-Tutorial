# Study Notes: Database Sharding & Partitioning

## Definitions

- **Partitioning**: Dividing a dataset into smaller, more manageable pieces. General umbrella term.
- **Vertical partitioning**: Splitting data by columns/tables/features (e.g., `users` table on one DB, `orders` on another; or splitting large/rarely-used columns into a separate table). Each partition has a different schema/subset of columns.
- **Horizontal partitioning (sharding)**: Splitting rows of a single logical table across multiple independent database instances ("shards"). Every shard shares the same schema but holds a different subset of rows.
- **Shard key (partition key)**: The column (or combination of columns) used to determine which shard a given row belongs to. The single most important design decision in a sharded system.
- **Replication vs. Sharding**: Replication copies the *same* full dataset to multiple nodes (helps availability + read scaling). Sharding splits *different* data across multiple nodes (helps write scaling + storage scaling). They are complementary, not substitutes — production systems often shard AND replicate each shard.

## Comparison Table: Sharding Strategies

| Criteria | Range-Based | Hash-Based | Directory-Based |
|---|---|---|---|
| Distribution evenness | Poor–moderate (depends on key distribution) | Good (near-uniform with a good hash fn) | Good (manually tunable/rebalanceable) |
| Range query support | Excellent (contiguous ranges map to one/few shards) | Poor (scattered across shards; needs fan-out + merge) | Moderate (depends on directory design) |
| Hotspot risk | High (sequential/time-based keys concentrate load on newest shard) | Low (pseudo-random spread) | Low–moderate (mitigated by manual rebalancing) |
| Resharding difficulty | Moderate (can split ranges, but data movement needed) | High with naive hash-mod-N (most keys remap); mitigated by consistent hashing | Low–moderate (just update directory mappings) |
| Operational complexity | Low–moderate | Low–moderate | High (extra lookup service to build, scale, and keep available) |

## Cross-Shard Query & Transaction Challenges

- **Joins**: Data related across shards can't be joined in a single SQL query; must be joined in application code by querying multiple shards and merging results.
- **Transactions**: A transaction spanning rows on two different shards can't rely on local ACID guarantees. Requires distributed transaction patterns — two-phase commit (2PC) or the Saga pattern (compensating transactions) — both add latency and complexity.
- **Aggregation/analytics**: Queries like "count all rows matching X across the whole dataset" require scattering the query to every shard and gathering ("scatter-gather") the results, which is slower and more resource-intensive than a single-shard query.
- **Secondary indexes**: An index on a non-shard-key column typically can't be maintained cleanly on a single shard; either duplicate the index across shards (scatter-gather lookups) or maintain a separate global index service.
- **Rebalancing**: Adding/removing shards requires moving data between them. Naive hash-mod-N remaps almost all keys on a resize; consistent hashing limits remapping to roughly `1/N` of the keys.

## Key Numbers / Rules of Thumb

- Naive `hash(key) % N` resharding: changing N from, say, 4 to 5 shards can require remapping up to ~80% of keys. Consistent hashing typically limits this to roughly `1/N` of keys moving.
- A common trigger point for considering sharding: a single primary instance can no longer keep up with write throughput, or the working data set no longer fits comfortably in memory/on a single disk, even after indexing and replication are already in place.
- Shard counts are often chosen as powers of two (e.g., 16, 32, 64) to make future splits (halving/doubling) cleaner.

## Summary Bullets

- Sharding scales writes and storage; replication scales reads and availability — most large systems need both.
- Vertical partitioning splits by column/table; horizontal partitioning (sharding) splits by row across instances.
- Range sharding: easy range queries, hotspot-prone.
- Hash sharding: even distribution, poor range queries.
- Directory-based sharding: most flexible, extra hop + infra cost.
- Consistent hashing minimizes data movement on resharding (deep dive covered separately).
- Choose a shard key with high cardinality, alignment to dominant query patterns, and low hotspot risk.
- Sharding's biggest costs are cross-shard joins/transactions, rebalancing effort, and multiplied operational overhead.
