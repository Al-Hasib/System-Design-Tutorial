# LSM Trees vs. B-Trees: Storage Engine Internals

**Difficulty:** Advanced
**Estimated length:** 14-18 min
**Prerequisites:** [12 - Database Indexing Explained](../../Module-03-Databases-and-Storage/12-database-indexing-explained/README.md), [11 - SQL vs NoSQL](../../Module-03-Databases-and-Storage/11-sql-vs-nosql/README.md)

## Learning Objectives

- Recall how a B-tree index works and why it's optimized for the access pattern of traditional relational databases.
- Explain what an LSM (Log-Structured Merge) tree is and how it turns random writes into sequential ones.
- Describe compaction and why it's necessary for an LSM tree's long-term health.
- Compare read, write, and space amplification trade-offs between B-trees and LSM trees.
- Choose the right storage engine family for a given workload's read/write ratio.

## Script

### Hook / Intro

Back in Module 3, we covered indexing at a conceptual level: B-trees for range queries, hash indexes for exact lookups. That's true, but it quietly assumes every database uses a B-tree — and a huge portion of the databases you'll actually design around in a system design interview, from Cassandra to RocksDB to the engine underneath many time-series and logging systems, don't. They use something called an LSM tree instead. Today we open up why: what B-trees are actually optimized for, what problem LSM trees solve that B-trees structurally can't solve well, and how to reason about which one your workload actually wants.

### B-Trees: Optimized for In-Place Updates

A B-tree index organizes data in a balanced tree of fixed-size pages (often matching the disk's block size), where each page holds sorted keys and pointers to child pages. To find a row, you walk down the tree — a handful of page reads gets you to any row, which is why B-trees give excellent, predictable read performance, including range scans, since sorted keys sit near each other on disk.

The cost shows up on writes. When you insert or update a row, the B-tree has to find the correct page and modify it *in place* — and that page could be anywhere on disk. At scale, with writes scattered across many different keys, this means a constant stream of random disk I/O: seek to this page, write it, seek to a completely different page, write it again. Random I/O is dramatically slower than sequential I/O, especially on spinning disks, and even on SSDs it still costs more than sequential writes due to how flash storage handles small random writes internally. B-trees also sometimes need to split a full page into two, rebalancing the tree — more overhead on the write path. This is exactly why B-tree-based databases (PostgreSQL, MySQL/InnoDB, most traditional relational databases) are excellent at read-heavy and range-query-heavy workloads, but start to strain under extremely high, sustained write volume.

### LSM Trees: Turning Random Writes into Sequential Ones

A Log-Structured Merge tree takes a completely different approach, built around one core idea: **never modify data in place — only ever append**. When a write comes in, it's first written to an in-memory structure (commonly called a **memtable**, often backed by a sorted structure like a skip list) and to an on-disk write-ahead log for durability in case of a crash. Both of these are sequential append operations — no seeking around the disk to find the right page. Once the memtable fills up, it's flushed to disk as an immutable, sorted file (often called an **SSTable** — Sorted String Table). Over time, many of these immutable SSTables accumulate on disk.

This design makes writes extremely fast, because every write is just an append — sequential I/O, no page-splitting, no in-place seek-and-modify. The trade-off shows up on reads: since the same key might exist in the memtable and in several different SSTables (an update just appends a newer version rather than overwriting the old one), a read might have to check multiple places and return the newest version it finds. Databases mitigate this with in-memory Bloom filters (letting a read skip an SSTable entirely if the filter says the key definitely isn't there) and by periodically running **compaction** — a background process that merges multiple SSTables together, discarding old, superseded versions of updated or deleted keys, and producing fewer, larger, still-sorted files. Compaction is what keeps read performance and disk usage from degrading indefinitely as more SSTables accumulate — it's ongoing background work that LSM-tree databases must budget CPU and I/O for.

### The Trade-off, Named Precisely

This comes down to three kinds of "amplification," a vocabulary worth having cold for an interview: **write amplification** — how many actual bytes get written to disk for each logical byte the application wrote (B-trees can have high write amplification from page rewrites and splits; LSM trees have lower write-path amplification but pay it back later during compaction). **Read amplification** — how many disk reads are needed to answer one logical read (B-trees: typically low and predictable, a few page reads; LSM trees: potentially several SSTables checked, mitigated by Bloom filters and compaction). **Space amplification** — how much extra disk space is used beyond the logical data size (LSM trees temporarily store multiple versions of updated/deleted data until compaction cleans it up; B-trees generally don't have this issue since updates happen in place). No storage engine minimizes all three simultaneously — every LSM-tree and B-tree implementation is a specific point on this trade-off triangle, tuned for its target workload.

### Real-World Example

Think about the difference between a traditional relational database backing an e-commerce product catalog (moderate write volume, heavy on range queries and joins — a B-tree engine like PostgreSQL's fits naturally) versus a time-series metrics store ingesting millions of data points per second from thousands of servers (extremely high, append-mostly write volume, where you mostly query recent data — an LSM-tree engine like Cassandra's or InfluxDB's underlying storage fits naturally). Or consider Cassandra directly: it's built entirely around LSM trees specifically because its design goal is to absorb massive write throughput across a distributed cluster, and it accepts the read-side complexity (multiple SSTables, Bloom filters, compaction) as the price for that write performance — exactly matching the write-heavy, append-heavy workloads it's usually chosen for.

### Recap

B-trees update data in place, giving excellent, predictable read and range-query performance at the cost of random-write I/O and page-splitting overhead — the right fit for read-heavy, moderate-write relational workloads. LSM trees never modify data in place — they only append, turning writes into fast sequential I/O — at the cost of needing to check multiple files on read (mitigated by Bloom filters) and requiring ongoing background compaction to keep both read performance and disk usage in check. Reason about your workload's actual read/write ratio and access pattern before assuming "a database" means "a B-tree" — a huge share of modern NoSQL and time-series systems are built on LSM trees for exactly this reason.

### What's Next

We've now gone deep on two pillars of database internals: how transactions stay isolated, and how the storage engine underneath actually persists data. Let's shift gears — next video looks at an API architecture question we've deferred since Module 2: what GraphQL actually solves that REST structurally can't, and where it earns its added complexity.

## Key Takeaways

- B-trees update data in place, giving fast, predictable reads and range scans, but pay for it with random-write I/O and page-splitting overhead on writes.
- LSM trees never modify data in place — writes are sequential appends to a memtable and write-ahead log, later flushed to immutable, sorted SSTables on disk.
- LSM trees trade read-path complexity (checking multiple SSTables, mitigated by Bloom filters) and ongoing background compaction for dramatically faster sustained write throughput.
- Write, read, and space amplification are the three axes every storage engine trades off against each other — no engine minimizes all three at once.
- Choose based on workload: read-heavy/range-query-heavy favors B-trees (most relational databases); write-heavy/append-heavy favors LSM trees (Cassandra, RocksDB, most time-series stores).
