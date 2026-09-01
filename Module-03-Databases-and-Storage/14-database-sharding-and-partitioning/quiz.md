# Practice & Interview Questions

**1. What is the difference between sharding and partitioning?**
Partitioning is the general concept of splitting data into smaller pieces, and can be vertical (by column/table) or horizontal (by row). Sharding specifically refers to horizontal partitioning where the resulting pieces ("shards") live on separate, independent database instances. So all sharding is partitioning, but not all partitioning is sharding.

**2. Why doesn't replication alone solve the problem of scaling writes or dataset size?**
In most replication setups (e.g., master-slave), every write still has to go through a single primary node, so adding more replicas doesn't add write capacity. Also, every replica typically stores a full copy of the entire dataset, so replication doesn't reduce how much data any single node has to hold — it just duplicates it.

**3. Explain range-based sharding and its main weakness.**
Range-based sharding assigns contiguous ranges of the shard key to each shard (e.g., IDs 1–1M on shard 1, 1M–2M on shard 2). It supports efficient range queries since a range scan usually touches only one or a few shards. Its main weakness is hotspotting: if the key correlates with time or is sequentially increasing, new/recent data — and often the most active data — piles onto one shard while older shards sit relatively idle.

**4. Explain hash-based sharding and its main weakness.**
Hash-based sharding runs the shard key through a hash function (often combined with modulo the shard count) to decide which shard a row lives on. This spreads data and load very evenly because a good hash function distributes keys pseudo-randomly. The weakness is that range queries become expensive — sequential keys are scattered across shards, so a range query must fan out to every shard and merge results.

**5. What is directory-based sharding, and when would you choose it over range or hash sharding?**
Directory-based sharding uses an explicit lookup service that maps each key (or key range) to a specific shard, instead of computing the mapping with a formula. You'd choose it when you need maximum flexibility — the ability to move individual keys between shards, rebalance unevenly loaded shards on the fly, or support heterogeneous shard capacities — and can tolerate the added complexity and extra network hop of consulting the directory on every query.

**6. What is consistent hashing, and why is it relevant to sharding?**
Consistent hashing is a hashing technique that minimizes how many keys need to move when you add or remove shards, unlike naive `hash(key) % N`, where changing N remaps nearly every key. It's relevant because resharding — adding capacity as you grow — is one of the most operationally painful parts of running a sharded system, and consistent hashing makes that process far less disruptive (typically only about `1/N` of keys move).

**7. What makes a good shard key? What should you avoid?**
A good shard key has high cardinality (many distinct values so data spreads across shards), aligns with your dominant query patterns (so most queries can be routed to a single shard), and avoids concentrating traffic on a small number of values. Avoid keys that are sequential/time-correlated (causes hotspots on the newest shard) or that have skewed real-world distributions, such as a "celebrity" user or a single dominant tenant receiving disproportionate traffic.

**8. What challenges does sharding introduce that a single-database system doesn't have?**
Cross-shard joins and transactions become hard, since data related across shards can't be joined or committed atomically without distributed transaction coordination (e.g., two-phase commit or sagas). Rebalancing data as shards grow unevenly is operationally difficult. And overall operational complexity increases — you now monitor, back up, and migrate schemas across N databases instead of one.

**9. How would you shard a table with 500 million rows that's outgrowing a single Postgres instance?**
First, identify the dominant access pattern — e.g., if most queries filter by `customer_id` or `tenant_id`, that's a strong shard-key candidate. Choose a hash-based scheme on that key (via an extension like Citus, or manually) to get even distribution and avoid hotspots, using enough shards to leave headroom for growth. Plan the migration as an online, incremental data move (dual-write or CDC-based backfill) rather than a single cutover, and design the application/router layer to direct single-tenant queries to one shard while accepting that cross-tenant analytics will need scatter-gather.

**10. What shard key would you pick for a multi-tenant SaaS application, and why?**
`tenant_id` (or `organization_id`) is usually the best choice, because nearly every query in a multi-tenant system is already scoped to a single tenant — pulling their users, records, and settings. Sharding on `tenant_id` means the vast majority of queries hit exactly one shard, keeping latency low and avoiding cross-shard joins, while data naturally isolates per customer, which also simplifies compliance and noisy-neighbor containment (assuming no single tenant is disproportionately huge relative to others).

**11. Your hash-sharded system needs to answer "how many total orders were placed last week across all customers." How would you handle this efficiently?**
Since the shard key doesn't align with this cross-cutting analytical query, you'd typically fan the query out to all shards in parallel (scatter-gather) and aggregate the partial counts in the application layer. For frequently-needed aggregates, a better long-term approach is to maintain a separate analytics store (e.g., a data warehouse or pre-aggregated rollup table) fed by a change-data-capture pipeline, so you're not hitting every operational shard for every reporting query.

**12. Compare the resharding difficulty of range-based, naive hash-based, and consistent-hash-based sharding.**
Range-based sharding can often be rebalanced by splitting a range into two, requiring you to move only the data in the split range. Naive hash-based sharding (`hash % N`) is the worst case: changing N remaps the great majority of keys, forcing a massive data migration. Consistent hashing solves this specific problem within the hash-based approach — adding or removing a node only reassigns roughly `1/N` of the keys, making it the preferred technique for hash-sharded systems that need to scale shard count over time.
