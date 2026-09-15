# Why This Topic Matters: Database Indexing

> **In one sentence:** Indexing is the single highest-leverage performance fix in most systems — a one-line change that turns a 30-second query into a 3-millisecond one — and the reason most "we need to scale the database" conversations are premature.

## The World Before This Idea

Without an index, finding one row means reading every row. That's fine at 1,000 rows and catastrophic at 50 million.

The failure mode is insidious because it's invisible during development. A query against a seed dataset of 200 rows returns instantly whether or not an index exists. The same query in production, six months later, against 20 million rows, takes 40 seconds and saturates the database CPU — and because it's slow, connections pile up waiting for it, the connection pool exhausts, and *unrelated* queries start failing too. A missing index on one column takes down a whole application.

## The Problems It Solves

### 1. Linear scans that scale with your success
**What you see:** Response times that were fine at launch degrade steadily and then fall off a cliff. Nothing changed in the code.

**Why it happens:** A full table scan is O(n). As the table grows, the query gets proportionally slower, until it crosses the threshold where it no longer fits in memory or no longer completes within the timeout.

**How indexing solves it:** A B-tree lookup is O(log n). Going from 1 million to 1 billion rows takes a B-tree from roughly 3 levels to roughly 5 — a 1000x increase in data for less than 2x the work. That's the difference between a system that survives growth and one that doesn't.

### 2. One slow query poisoning the entire database
**What you see:** Checkout starts failing. Investigation shows checkout is fine — an analytics query with no index is holding connections and starving everything else.

**Why it happens:** A database has finite connections, memory, and I/O bandwidth. A scan of a huge table consumes all three and evicts everyone else's cached pages from the buffer pool as a bonus.

**How indexing solves it:** Making the offending query cheap removes the resource contention. This is why "add an index" often fixes symptoms that look nothing like a query problem.

### 3. Sorting and range queries that can't be optimized away
**What you see:** `ORDER BY created_at DESC LIMIT 20` on a large table is slow even though it returns only 20 rows.

**Why it happens:** Without an ordered index, the database must read and sort *everything* before it can know which 20 are the latest.

**How indexing solves it:** A B+Tree stores values in sorted order with linked leaves, so the database can walk straight to the right end of the range and read 20 entries. This is exactly why B-trees dominate over hash indexes despite hashing being faster for pure equality: real applications sort and range-scan constantly.

### 4. "Just add an index to everything" making writes collapse
**What you see:** After a performance push added indexes everywhere, read latency improved and write throughput fell by half.

**Why it happens:** Every index must be updated on every insert, update, and delete. Eight indexes means one logical write becomes nine physical writes, plus page splits and rebalancing.

**How this topic solves it:** It reframes indexing as a *trade* — read speed purchased with write cost and storage — so you index deliberately: high-cardinality columns that are actually filtered, joined, or sorted on, and nothing else.

## The Price You Pay

- **Write amplification.** Covered above, and it's the main one. Write-heavy tables (events, logs, metrics) should carry the minimum viable index set.
- **Storage.** Indexes are real data structures. A heavily indexed table can consume more space in indexes than in rows.
- **Maintenance.** Fragmentation, bloat, and stale statistics degrade index effectiveness over time. And building an index on a huge live table can lock it — an operational hazard that requires online/concurrent index builds.
- **Indexes the planner won't use.** A composite index on `(a, b)` does nothing for a query filtering only on `b`. An index on a low-selectivity column gets ignored because a scan is genuinely cheaper. Creating indexes you believe are helping while they only cost you is common.

## When You Need It — and When You Don't

| Index when | Don't index when |
|---|---|
| The column appears in `WHERE`, `JOIN`, or `ORDER BY` | The column is never filtered or sorted on |
| Cardinality is high (email, user ID, timestamp) | Cardinality is low (booleans, three-value status) |
| The table is large and read-heavy | The table is tiny — a scan is already fast |
| A query plan confirms a sequential scan is the bottleneck | The table is write-dominated and reads are rare |

## Why This Shows Up in Interviews

"This query is slow — what do you do?" is a standard question, and the expected process is: look at the query plan, identify the scan, add the right index, verify. Deeper follow-ups probe whether you understand the *structure*: why B-trees support ranges and hash indexes don't, why composite index column order matters, what a covering index buys you, and what indexing costs on the write path. Candidates who mention the write trade-off unprompted stand out, because it shows they've operated a database rather than only queried one.

## How It Connects

Indexing is the first thing to try before reaching for heavier machinery — most systems that "need sharding" actually need an index. It builds directly on the **SQL vs NoSQL** choice (NoSQL systems have their own index models and constraints), underpins **caching** decisions (an indexed query may be fast enough that a cache isn't worth the invalidation complexity), and connects to **LSM trees vs B-trees**, which explains the storage-engine level beneath it.

**Next:** [Database Replication](../13-database-replication/why.md) — what to do when one database node isn't enough, or isn't safe enough.
