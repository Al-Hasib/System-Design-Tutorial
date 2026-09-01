# Study Notes: LSM Trees vs. B-Trees

## Definitions

- **B-tree:** A balanced tree of fixed-size pages storing sorted keys; updates happen in place, requiring a seek to the relevant page.
- **LSM (Log-Structured Merge) tree:** A storage structure where writes are appended to an in-memory memtable and a write-ahead log, then flushed as immutable, sorted SSTables — never modified in place.
- **Memtable:** The in-memory, sorted structure that buffers recent writes before they're flushed to disk.
- **SSTable (Sorted String Table):** An immutable, sorted on-disk file produced when a memtable is flushed.
- **Write-ahead log (WAL):** An append-only log written before/alongside the memtable update, for crash recovery.
- **Compaction:** Background process merging multiple SSTables into fewer, larger ones, discarding superseded/deleted data.
- **Write amplification:** Ratio of actual bytes written to disk vs. logical bytes the application wrote.
- **Read amplification:** Number of disk reads needed to satisfy one logical read.
- **Space amplification:** Extra disk space used beyond the logical size of the data.

## B-Trees vs. LSM Trees

| Aspect | B-Tree | LSM Tree |
|---|---|---|
| Write pattern | In-place update (random I/O) | Append-only (sequential I/O) |
| Write throughput | Moderate | High |
| Read pattern | Direct page traversal (fast, predictable) | May check memtable + multiple SSTables |
| Read throughput | High, predictable | Can be slower, mitigated by Bloom filters |
| Range queries | Excellent (data sorted in place) | Good, but may need to merge across SSTables |
| Background maintenance | Occasional page splits/rebalancing | Ongoing compaction |
| Typical users | PostgreSQL, MySQL/InnoDB, most RDBMSs | Cassandra, RocksDB, LevelDB, HBase, many time-series DBs |
| Best fit | Read-heavy, range-query-heavy workloads | Write-heavy, append-heavy, high-ingest workloads |

## Amplification Trade-off Triangle

| Engine | Write amplification | Read amplification | Space amplification |
|---|---|---|---|
| B-tree | Moderate-high (page rewrites/splits) | Low | Low |
| LSM tree (uncompacted) | Low on write path | High (many SSTables to check) | High (old versions not yet cleaned) |
| LSM tree (after compaction) | Higher (data rewritten during compaction) | Lower (fewer SSTables) | Lower (superseded data discarded) |

No engine minimizes all three simultaneously — compaction strategy tunes where an LSM tree sits on this triangle.

## How LSM Tree Writes and Reads Work

1. **Write:** Append to write-ahead log (durability) → insert into in-memory memtable (sorted).
2. **Flush:** When memtable fills, flush it as a new immutable SSTable on disk.
3. **Read:** Check memtable first, then check SSTables newest-to-oldest (Bloom filters skip SSTables that definitely don't contain the key), return the newest version found.
4. **Compaction:** Periodically merge multiple SSTables, dropping obsolete/deleted versions, producing fewer larger sorted files.

## Key Numbers / Facts

- Sequential disk I/O can be an order of magnitude (or more) faster than random I/O, especially on spinning disks — the core reason LSM trees favor append-only writes.
- Cassandra, RocksDB, LevelDB, and HBase are all built on LSM-tree storage engines.
- PostgreSQL and MySQL/InnoDB (the two most widely used open-source relational databases) both use B-tree indexes as their primary storage structure.

## Summary

- B-trees optimize for fast, predictable reads and range scans by updating data in place — at the cost of random-write I/O.
- LSM trees optimize for fast writes by only ever appending — at the cost of read-path complexity (multiple SSTables) and the need for ongoing compaction.
- Match the storage engine family to the workload's actual read/write ratio rather than assuming one universal "database" internal structure.
