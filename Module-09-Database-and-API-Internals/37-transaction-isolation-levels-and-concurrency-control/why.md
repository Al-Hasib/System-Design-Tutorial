# Why This Topic Matters: Transaction Isolation Levels & Concurrency Control

> **In one sentence:** Your database has a default isolation level that you almost certainly did not choose, and it permits a specific set of anomalies that will eventually produce a bug you cannot reproduce — because it only happens when two users act at the same instant.

## The World Before This Idea

The classic case. Code reads a balance, checks it is sufficient, subtracts, and writes it back:

```
balance = SELECT balance FROM accounts WHERE id = 1   -- reads 100
if balance >= 100: UPDATE accounts SET balance = 0 WHERE id = 1
```

Two withdrawals of 100 run simultaneously against a balance of 100. Both read 100. Both pass the check. Both write 0. The account has paid out 200. This is a **lost update**, and the code looks completely correct in review.

Now the harder part: whether this can happen depends on the isolation level. Under Read Committed — the default in PostgreSQL, Oracle, and SQL Server — it *can*. Under Repeatable Read in PostgreSQL, the second transaction gets a serialization error instead. Same code, different outcome, decided by a setting most teams never look at.

## The Problems It Solves

### 1. Anomalies you cannot reproduce
**What you see:** Duplicate bookings, negative inventory, double-spent credits, a report whose numbers do not add up. It happens once a week in production and never in testing.

**Why it happens:** These are timing-dependent. They require two transactions to interleave in a particular way, which is rare at low concurrency and constant at high concurrency. Test suites run serially and never produce them.

**How isolation levels solve it:** They give the anomalies names and a defined containment. **Dirty read** (seeing uncommitted data), **non-repeatable read** (the same row changes within your transaction), **phantom read** (new rows appear matching your query), and **lost update** each have a level at which they are prevented. Once you know which anomalies your level permits, "can this bug happen?" becomes a question you can answer from the documentation instead of from production.

### 2. Check-then-act races that look correct
**What you see:** Code that reads, decides, and writes — the most natural pattern in programming — producing wrong results under load.

**Why it happens:** The gap between the read and the write is where another transaction slips in.

**How concurrency control solves it:** Either take a lock at read time (`SELECT ... FOR UPDATE` — pessimistic), or detect the conflict at write time via a version number (`UPDATE ... WHERE version = 7` — optimistic), or push the decision into the database as an atomic operation (`UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100`). Knowing that the third option exists, and preferring it, removes whole categories of race with no locking at all.

### 3. Readers and writers blocking each other
**What you see:** A long analytical query holds shared locks and every write behind it stalls; or a write holds an exclusive lock and every reader waits.

**Why it happens:** Pure lock-based concurrency control makes readers and writers mutually exclusive.

**How MVCC solves it:** Multi-Version Concurrency Control keeps multiple versions of each row. A reader sees a consistent snapshot as of the moment its transaction began, without taking locks; a writer creates a new version without blocking readers. **Readers never block writers, and writers never block readers.** This is why PostgreSQL, MySQL InnoDB, and Oracle all use MVCC, and it is one of the highest-impact ideas in database engineering.

### 4. Correctness bought at the cost of throughput
**What you see:** Switching everything to Serializable to be safe, and watching throughput collapse under transaction retries and lock contention.

**Why it happens:** Serializable is the strongest guarantee and the most expensive — either heavy locking or frequent serialization failures that must be retried.

**How this topic solves it:** It lets you scope the guarantee. Run most of the application at the default level, and raise isolation (or take an explicit lock) only for the specific transactions where an anomaly would be costly. Correctness per operation, not per system — the same principle that appears in consistency models.

## The Price You Pay

- **Higher isolation costs concurrency.** More locking, more blocking, more aborted transactions to retry. There is no free correctness.
- **Optimistic concurrency requires retry logic everywhere it is used.** Under high contention, retries can dominate and throughput can be worse than pessimistic locking.
- **Pessimistic locking risks deadlocks.** Two transactions taking the same locks in different orders will deadlock; the database detects it and kills one, which your application must handle. Consistent lock ordering is the standard prevention, and it is a discipline that must be maintained across a whole codebase.
- **MVCC generates garbage.** Old row versions accumulate and must be cleaned up. In PostgreSQL this is `VACUUM`, and a long-running transaction that prevents cleanup causes table bloat — a real and common production problem that surprises teams the first time.
- **The same level name means different things.** PostgreSQL's Repeatable Read prevents some anomalies that MySQL's does not; "Repeatable Read" in one engine is not the same guarantee as in another. Portable assumptions are unsafe.
- **None of this crosses databases.** Isolation is per-database. As soon as an operation spans services or shards, you are back in the world of sagas and idempotency.

## When You Need It — and When You Don't

| Raise isolation or lock explicitly when | Defaults are fine when |
|---|---|
| Money, inventory, seats, or quotas are involved | The operation is a simple insert or a single-row read |
| The logic is read-check-write | Reads are for display and slight staleness is fine |
| A uniqueness or capacity invariant must hold | The write is naturally idempotent and order-independent |
| A report must be internally consistent | Contention is genuinely low and stakes are low |

**Best first move:** rewrite read-check-write as a single atomic statement or rely on a database constraint. That is cheaper and safer than any isolation level discussion.

## Why This Shows Up in Interviews

This is a senior-level differentiator. Prompts involving ticket booking, seat reservation, inventory, or account balances are concurrency questions in disguise, and the interviewer is waiting to see whether you notice. Saying "two users could book the last seat simultaneously, so I would either take a row lock with `SELECT FOR UPDATE`, use optimistic concurrency with a version column, or make it a conditional atomic update — and here is why I would pick one" is a very strong answer. Knowing what MVCC is, and that readers do not block writers, demonstrates you understand what is happening beneath the SQL.

## How It Connects

This is the "I" in **ACID** (topic 16), examined closely. It is the single-node counterpart to the **consistency models** of topic 29 — the same underlying tension between correctness and concurrency, one inside a database and one across a network. It stops working across **shards** (topic 14) and **microservices** (topic 30), which is exactly why **distributed transactions** (topic 28) and **distributed locking** (topic 40) exist. MVCC's implementation ties into **storage engine internals** (topic 38), and it is decisive in the **ride-sharing** (topic 54) and booking-style case studies.

**Next:** [LSM Trees vs B-Trees](../38-lsm-trees-vs-b-trees-storage-engine-internals/why.md) — one layer deeper, into how rows are actually written to disk.
