# Why This Topic Matters: LSM Trees vs B-Trees — Storage Engine Internals

> **In one sentence:** "Cassandra is fast at writes and Postgres is fast at reads" is a claim most engineers repeat and cannot explain — the explanation is the storage engine, and knowing it turns database selection from folklore into reasoning.

## The World Before This Idea

You benchmark two databases on the same hardware with the same data. One ingests 200,000 writes per second; the other manages 20,000. The second answers range queries instantly; the first sometimes takes far longer than its average would suggest. Neither is better-engineered. They made opposite choices at the layer where data meets disk.

Without that layer, database choice is driven by benchmarks you did not design and blog posts written about someone else's workload. You cannot predict how a system will behave on *your* access pattern, and you cannot explain the latency spike that shows up in production three months in.

## The Problems It Solves

### 1. Random writes that disks are bad at
**What you see:** A write-heavy workload — events, metrics, logs, sensor data — saturating disk I/O far below the hardware's rated throughput.

**Why it happens:** A B-Tree updates data *in place*. Writing a row means finding its page, reading it, modifying it, and writing it back — a random I/O for every write, plus occasional page splits. Random I/O is where storage devices are weakest, especially spinning disks, and still meaningfully worse than sequential on SSDs.

**How LSM trees solve it:** Writes go to an in-memory structure and an append-only write-ahead log, then are flushed to disk as immutable sorted files, entirely sequentially. Sequential writes are dramatically faster, and there is no read-modify-write cycle. This is the whole reason Cassandra, RocksDB, LevelDB, HBase, and ScyllaDB exist and why they ingest so much more than B-Tree engines.

### 2. Reads that must check many places
**What you see:** In an LSM store, a lookup for a key that does not exist is unexpectedly expensive, and read latency varies more than you would like.

**Why it happens:** Data for one key may live in the memtable or in any of several on-disk levels. A read may have to check all of them — the cost of never updating in place.

**How the engine solves it:** Bloom filters in front of each file answer "definitely not here" in constant time, eliminating most unnecessary disk reads. Sparse indexes and block caches handle the rest. Understanding this explains why Bloom filters are not an exotic optimization but a load-bearing part of every LSM engine — and why LSM read amplification is manageable but never zero.

### 3. Unpredictable latency spikes in production
**What you see:** p99 latency that is ten or a hundred times p50, appearing in bursts with no traffic correlation.

**Why it happens:** **Compaction.** LSM engines must periodically merge sorted files to reclaim space from deleted and overwritten rows and to limit how many files a read must check. Compaction is I/O-heavy background work that competes with live traffic. This is the single most important operational fact about LSM stores, and it is why they can look excellent in a short benchmark and behave differently after weeks of production writes.

**How this topic solves it:** It tells you the spike is inherent, not a bug, and points at the controls — compaction strategy (size-tiered favors write throughput, leveled favors read performance and space efficiency), throughput throttling, and provisioning headroom for it.

### 4. Space that does not come back after deletes
**What you see:** You delete half the rows and disk usage does not drop. In some engines it briefly increases.

**Why it happens:** LSM deletes are **tombstones** — markers written as new data. The original row persists until compaction removes both. In B-Trees, the mirror problem is different but real: deleted space becomes free pages inside the file, which the OS does not reclaim, causing bloat that needs `VACUUM` or a rebuild.

**How this topic solves it:** It makes storage behavior predictable. Knowing about tombstones also explains a notorious Cassandra failure mode: queuing many deletes and then range-scanning that region forces the engine to read through enormous numbers of tombstones, and queries time out.

## The Price You Pay

Each engine pays a different tax, and the honest framing is that you are choosing *which* amplification to pay:

- **B-Trees pay write amplification.** Every write is a random in-place page update plus a write-ahead log entry, so one logical write is several physical ones. In exchange: predictable low-variance reads, efficient range scans over sorted pages, and straightforward locking for strong transactional guarantees.
- **LSM trees pay read amplification and space amplification.** Reads may check several levels; deleted and overwritten data occupies space until compaction. In exchange: very high sequential write throughput and good compression, since immutable sorted blocks compress well.
- **Compaction tuning is a real operational specialty.** Choosing and tuning a strategy for your workload is ongoing work, not a one-time setting.
- **Neither is a general answer.** A workload that is read-heavy with lots of range scans and updates in place is a B-Tree workload. A workload that is append-dominated with key lookups is an LSM workload. Most systems have both, in different tables — which is an argument for choosing per-dataset rather than per-company.

## When You Need It — and When You Don't

| LSM (Cassandra, RocksDB, ScyllaDB, HBase) when | B-Tree (Postgres, MySQL InnoDB, SQL Server) when |
|---|---|
| Write volume is very high and append-like | Reads dominate, especially range scans |
| Time series, events, logs, metrics, sensor data | The workload is transactional OLTP |
| Storage efficiency and compression matter | Latency must be predictable and low-variance |
| You can absorb p99 variance from compaction | You need strong multi-row transactional guarantees |

## Why This Shows Up in Interviews

This is a strong differentiator at senior level. When asked "why Cassandra here?", the folklore answer is "it scales." The engineering answer is: "the workload is append-heavy time series with key-based reads, and an LSM engine turns those writes into sequential I/O, which is where the throughput comes from — at the cost of read amplification, which Bloom filters mitigate, and compaction, which I would provision headroom for." That answer demonstrates you can reason about a database rather than recite its marketing. It also explains **why** Bloom filters keep appearing in system design, which makes topic 42 feel inevitable rather than arbitrary.

## How It Connects

This is the layer beneath **indexing** (topic 12) and explains the deep reason for the performance profiles behind **SQL vs NoSQL** (topic 11). **Bloom filters** (topic 42) are a core component of LSM reads. **MVCC** (topic 37) is implemented on top of these structures. Write throughput characteristics feed directly into **sharding** decisions (topic 14), and compaction-driven latency variance is something **observability** (topic 43) has to surface before it surprises you.

**Next:** [GraphQL: A Query-Based Alternative to REST](../39-graphql-a-query-based-alternative-to-rest/why.md) — back up to the API layer, and a different way of asking for data.
