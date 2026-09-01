# Practice & Interview Questions

**1. What is a dirty read, and which isolation level allows it?**
A dirty read occurs when a transaction reads another transaction's uncommitted changes. If that other transaction later rolls back, the reader has acted on data that never actually existed. Only Read Uncommitted allows dirty reads; Read Committed and above prevent them.

**2. Explain the difference between a non-repeatable read and a phantom read.**
A non-repeatable read is reading the same row twice within a transaction and getting different values because another transaction committed a change to that specific row in between. A phantom read is re-running the same query (not a single row) and getting a different set of matching rows because another transaction inserted or deleted rows matching the condition in between.

**3. Why would a system choose Read Committed over Serializable, given that Serializable prevents more anomalies?**
Serializable isolation requires far more coordination — extensive locking or conflict detection — which reduces throughput and increases the rate of aborted/retried transactions. Read Committed is a pragmatic default that prevents dirty reads (the most dangerous anomaly) while preserving much higher concurrency for workloads that can tolerate the remaining, rarer anomalies.

**4. Compare pessimistic locking and optimistic concurrency control, and describe a scenario where each is the better choice.**
Pessimistic locking acquires a lock before touching data, preventing conflicts upfront at the cost of contention and possible deadlock — a good fit for a hot, high-contention row like a limited inventory count. Optimistic concurrency control lets transactions proceed without locking and only checks for conflicts at commit time, retrying if one occurred — a good fit for typical low-contention data where conflicts are rare and locking overhead would be wasted most of the time.

**5. What problem does MVCC solve that pure locking doesn't?**
Pure locking can force readers to block behind writers (and vice versa) even when a reader just wants a consistent snapshot, not to modify anything. MVCC keeps multiple versions of each row so a reader can see a consistent, committed snapshot without ever blocking on a concurrent writer, and a writer never has to wait for a concurrent reader to finish.

**6. Why does an MVCC-based database like PostgreSQL need a process like `VACUUM`?**
Because MVCC keeps old row versions around so that transactions that started before a change was committed can still see the version consistent with their own start time. Once no active transaction needs an old version anymore, it becomes dead space that must be reclaimed — `VACUUM` is PostgreSQL's background process for cleaning up these old versions and preventing table bloat.

**7. Scenario: Two customers try to book the last available seat on a flight at the same instant. Would you rely on your database's default isolation level and MVCC alone, or add something else? Why?**
For a high-stakes, low-tolerance-for-error scenario like this, relying purely on MVCC's snapshot isolation risks both transactions believing a seat is available based on their own consistent snapshot. It's safer to explicitly force pessimistic locking on that specific row (e.g., `SELECT ... FOR UPDATE`) so the second transaction is forced to wait and re-check availability after the first commits, rather than trusting optimistic retry logic for something this critical.

**8. What is a deadlock, and how does a database typically handle it?**
A deadlock occurs when two or more transactions each hold a lock that the other needs, so neither can proceed. Databases detect this condition (often via a wait-for graph) and resolve it by aborting one of the transactions (the "victim"), releasing its locks so the other can proceed; the aborted transaction's application code is expected to retry.

**9. True or False: Under Repeatable Read, phantom reads are guaranteed to be impossible per the SQL standard.**
False. The SQL standard's definition of Repeatable Read only guarantees no non-repeatable reads on individually-read rows — phantom reads (new rows matching a range condition) are technically still permitted at this level per the standard, though some database engines' actual implementations (like MySQL's InnoDB) prevent many phantom scenarios through additional mechanisms like gap locking.

**10. Why does optimistic concurrency control require the application layer to implement retry logic, while pessimistic locking generally doesn't?**
With OCC, a conflict is only discovered at commit time, and the transaction is aborted rather than automatically resolved — the application must detect this failure and decide whether/how to retry the entire operation. With pessimistic locking, the database itself makes conflicting transactions wait until the lock is free, so the operation naturally proceeds in order without needing the application to explicitly retry.
