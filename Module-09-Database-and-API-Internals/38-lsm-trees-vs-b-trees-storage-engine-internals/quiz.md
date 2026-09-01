# Practice & Interview Questions

**1. Why do B-tree writes require random disk I/O while LSM tree writes are sequential?**
A B-tree updates a row in place, which means finding and modifying whatever page on disk currently holds that row — that page could be anywhere, so writes scatter across the disk (random I/O). An LSM tree never modifies existing data in place; every write is appended to the end of an in-memory memtable and a write-ahead log, which are always sequential operations regardless of which key is being written.

**2. What is a memtable, and what happens to it once it's full?**
A memtable is the in-memory, sorted structure that buffers recent writes in an LSM tree. Once it fills up, it's flushed to disk as a new immutable SSTable (Sorted String Table), and a fresh, empty memtable takes over new writes.

**3. Why might a single read in an LSM tree need to check multiple SSTables?**
Because updates and even deletes are implemented as new appended entries rather than in-place modifications, the same key can have entries in the memtable and in several different SSTables written at different times. A read has to check these (newest first) to find the most recent version of that key.

**4. What is compaction, and why is it necessary?**
Compaction is a background process that merges multiple SSTables into fewer, larger sorted files, discarding old versions of updated keys and any tombstoned (deleted) keys. Without it, reads would have to check an ever-growing number of SSTables (rising read amplification) and disk usage would grow unboundedly with stale data (rising space amplification).

**5. Define write amplification, read amplification, and space amplification.**
Write amplification is the ratio of actual bytes written to disk versus the logical bytes the application wrote. Read amplification is how many disk reads are needed to satisfy one logical read. Space amplification is how much extra disk space is used beyond the logical size of the data. Every storage engine trades these off against each other.

**6. Why do Bloom filters help LSM tree reads, and what's the trade-off of using one?**
A Bloom filter can quickly tell a reader that a key is *definitely not* present in a given SSTable, letting the read skip that file entirely without a disk access — reducing read amplification. The trade-off is that Bloom filters have a small false-positive rate (they can say "maybe present" for a key that isn't actually there, costing a wasted lookup), though they never produce false negatives.

**7. Scenario: You're choosing a storage engine for a system ingesting millions of sensor readings per second, mostly queried for recent data. Which storage engine family fits better, and why?**
An LSM-tree-based engine (like Cassandra's or a time-series database built on RocksDB/LevelDB) fits better — the workload is extremely write-heavy and append-mostly, which plays directly to LSM trees' strength of turning writes into fast sequential I/O, and the read pattern (mostly recent data) tends to hit recently-flushed SSTables that haven't accumulated much read amplification yet.

**8. Scenario: You're choosing a storage engine for an e-commerce product catalog with heavy filtering, sorting, and range queries, and moderate write volume. Which fits better, and why?**
A B-tree-based engine (like PostgreSQL or MySQL/InnoDB) fits better — the workload is read-heavy and range-query-heavy, which is exactly what B-trees are optimized for, and the moderate write volume doesn't push hard against B-trees' in-place-update cost the way a high-ingest workload would.

**9. Why can neglecting compaction eventually hurt both read performance and disk usage in an LSM-tree database?**
Without compaction, SSTables keep accumulating and old, superseded versions of updated/deleted keys are never cleaned up. Reads have to check a growing number of SSTables (higher read amplification, slower reads), and disk usage grows with stale data that's logically no longer needed (higher space amplification).

**10. True or False: A B-tree-based database can never handle high write throughput.**
False, but with an important caveat — B-trees can handle substantial write throughput, and real-world B-tree databases use techniques (write-ahead logs, buffering, batching) to improve write performance. The point is a *relative* one: for extremely high, sustained write volume, an LSM-tree engine's append-only design fundamentally avoids the random I/O and page-splitting cost that B-trees pay on every in-place update, which is why the highest-write-throughput systems tend to choose LSM trees.
