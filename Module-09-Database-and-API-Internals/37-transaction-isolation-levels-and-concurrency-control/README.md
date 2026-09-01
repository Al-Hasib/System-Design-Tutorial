# Transaction Isolation Levels & Concurrency Control: Locking vs. MVCC

**Difficulty:** Advanced
**Estimated length:** 16-20 min
**Prerequisites:** [16 - ACID vs BASE, Normalization vs Denormalization](../../Module-03-Databases-and-Storage/16-acid-vs-base-normalization-vs-denormalization/README.md), [13 - Database Replication](../../Module-03-Databases-and-Storage/13-database-replication/README.md)

## Learning Objectives

- Explain what a race condition looks like at the database level when two transactions touch the same data concurrently.
- Describe the four standard SQL isolation levels and which anomalies each one prevents.
- Compare pessimistic locking (lock, then act) with optimistic concurrency control (act, then check).
- Explain MVCC (Multi-Version Concurrency Control) and why it lets readers avoid blocking writers.
- Choose an appropriate isolation level and concurrency strategy for a given system design scenario.

## Script

### Hook / Intro

Back in Module 3, we defined the "I" in ACID as "Isolation" — concurrent transactions shouldn't interfere with each other — and moved on. That definition is true but does almost no work for you in an actual interview or an actual production incident. What does "shouldn't interfere" actually mean when two customers try to book the last seat on a flight at the exact same millisecond? Today we open up isolation for real: the specific anomalies that can occur when transactions overlap, the isolation levels that trade correctness for performance, and the two fundamentally different engineering approaches — locking and MVCC — that databases use to enforce whichever level you choose.

### The Core Problem: Concurrent Transactions

A transaction is a group of operations that should behave as one atomic unit. In a single-user world, transactions never overlap and there's nothing to reason about. The moment you have concurrent users — which is every real system — two transactions might read and write overlapping data at the same time, and without any protection, several specific anomalies can occur.

**Dirty read**: Transaction A reads a row that transaction B has modified but not yet committed. If B then rolls back, A has now acted on data that never actually existed. **Non-repeatable read**: Transaction A reads the same row twice, and gets different values each time, because B committed a change to that row in between A's two reads. **Phantom read**: Transaction A runs the same query twice (e.g., "count all orders over $100"), and gets a different set of rows the second time, because B inserted or deleted a row matching that condition in between. Each of these is a specific, well-defined way that concurrency can produce results that would never happen if transactions ran one at a time.

### The Four SQL Isolation Levels

The SQL standard defines four isolation levels, each permitting fewer of these anomalies at the cost of more coordination overhead:

**Read Uncommitted** — the weakest level. Transactions can see each other's uncommitted changes. Dirty reads are possible. Rarely used in practice; some databases (like PostgreSQL) don't even implement it distinctly from Read Committed.

**Read Committed** — the default in most production databases (PostgreSQL, SQL Server, Oracle). A transaction only ever sees data that's been committed by other transactions. Dirty reads are prevented, but non-repeatable reads and phantom reads can still happen, because each individual read within the transaction sees the latest committed state at that instant, which can change between reads.

**Repeatable Read** — MySQL's default (via InnoDB). Once a transaction reads a row, it's guaranteed to see the same value for that row for the rest of the transaction, even if another transaction commits a change to it. This prevents non-repeatable reads. Phantom reads are still technically possible in the SQL standard's definition, though several databases' actual implementations (like InnoDB's) prevent most phantom scenarios anyway.

**Serializable** — the strongest level. Transactions behave exactly as if they ran one at a time, in some serial order, even though they're actually running concurrently. All three anomalies are prevented. The cost is the highest: the database has to do the most work — extensive locking or aggressive conflict detection — to guarantee this, which reduces throughput and increases the odds of transactions having to abort and retry.

The pattern across all four levels is the same trade-off we've seen throughout this course: stronger correctness guarantees cost more coordination and throughput. Choosing an isolation level is an explicit trade-off decision, not a default you should leave unexamined.

### Pessimistic Locking vs. Optimistic Concurrency Control

There are two fundamentally different engineering strategies for actually enforcing isolation.

**Pessimistic locking** assumes conflicts are likely, so it prevents them upfront: before a transaction touches a row, it acquires a lock on it, and any other transaction that wants to touch the same row has to wait until the lock is released. This is simple to reason about and safe by construction, but it costs throughput — transactions queue up waiting for locks, and if a transaction holding a lock is slow (or worse, two transactions each hold a lock the other needs), you get contention or even **deadlock**, which the database has to detect and resolve by aborting one of the transactions.

**Optimistic concurrency control (OCC)** assumes conflicts are rare, so it lets transactions proceed without locking anything, and only checks for a conflict right before committing — typically by comparing a version number or timestamp on the row against what it was when the transaction started. If nothing changed, the commit succeeds. If something changed, the transaction is aborted and the application retries. This avoids the throughput cost of locking when conflicts really are rare, but wastes work (and requires retry logic in the application) whenever a conflict does occur. The right choice depends entirely on your actual contention pattern: a high-contention hot row (like a popular product's inventory count) favors pessimistic locking; a low-contention scenario (most rows, most of the time, in most applications) favors optimistic concurrency, which is why it's the default assumption in many ORMs' "optimistic locking" features.

### MVCC: How Modern Databases Actually Do This

Most production relational databases (PostgreSQL, MySQL/InnoDB, Oracle) don't rely purely on locking readers out — they use **Multi-Version Concurrency Control (MVCC)**. Instead of a row having one single current value that readers and writers fight over, the database keeps multiple versions of a row, each tagged with the transaction that created it. When a transaction reads a row, it doesn't see "the current value" — it sees the version that was committed as of when its own transaction (or statement) started. This means a long-running read never has to block a concurrent write, and a write never has to block a concurrent read, because they're literally looking at different versions of the same logical row. Writers still need to coordinate with other writers (you can't have two transactions both "win" a conflicting update to the same row), but the classic reader-vs-writer contention that pure locking creates mostly disappears. The cost is that old row versions need to be cleaned up eventually once no transaction still needs them — this is exactly what PostgreSQL's `VACUUM` process does, and why an application generating enormous numbers of updates without regular vacuuming can suffer from table bloat.

### Real-World Example

Consider our flight-booking scenario: two customers try to book the last seat at the same moment. Under Read Committed with pessimistic locking, the first transaction to reach the seat row locks it, decrements the available-seats counter, and commits; the second transaction, having waited for the lock, now sees zero seats available and fails cleanly — no double-booking. Under an MVCC-based Repeatable Read approach, both transactions might start by reading "1 seat available" from their own consistent snapshot, but the underlying write path still requires the second transaction's actual update to detect that the row changed since it read it, causing a serialization failure that the application must catch and retry — which is exactly why high-stakes financial and inventory systems generally can't just trust MVCC's isolation and walk away; they still need explicit conflict-handling logic (a `SELECT ... FOR UPDATE` to force pessimistic locking on exactly this row, for instance) at the specific points where correctness genuinely can't tolerate a race.

### Recap

Concurrent transactions can produce dirty reads, non-repeatable reads, and phantom reads if left unguarded. The four SQL isolation levels — Read Uncommitted, Read Committed, Repeatable Read, Serializable — trade off which of these anomalies are prevented against coordination overhead and throughput. Pessimistic locking prevents conflicts upfront at the cost of contention and possible deadlock; optimistic concurrency control lets transactions proceed and only checks for conflicts at commit time, favoring low-contention workloads. MVCC is how most modern databases actually implement isolation in practice — keeping multiple row versions so readers and writers don't block each other — but hot, high-stakes rows still often need explicit locking on top of it.

### What's Next

We've gone deep on how a database enforces correctness between concurrent transactions. Next video, we go one layer further down — into the actual data structure a database's storage engine uses to store and index that data on disk, and why write-heavy systems reach for a fundamentally different structure than the B-trees we introduced back in Module 3.

## Key Takeaways

- Unprotected concurrent transactions can produce dirty reads, non-repeatable reads, and phantom reads — each a specific, well-defined anomaly.
- The four SQL isolation levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable) prevent progressively more anomalies at progressively higher coordination cost.
- Pessimistic locking prevents conflicts upfront (safe, but costs throughput and risks deadlock); optimistic concurrency control checks for conflicts only at commit time (fast under low contention, requires retry logic).
- MVCC lets most modern databases avoid classic reader-vs-writer blocking by keeping multiple versions of each row, at the cost of needing background cleanup (e.g., PostgreSQL's `VACUUM`).
- Isolation level and concurrency strategy are explicit engineering trade-offs — high-contention, high-stakes rows (like inventory counts) often still need explicit locking even in an MVCC database.
