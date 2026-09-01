# Practice & Interview Questions

**1. Explain each ACID property with a one-line example.**
- Atomicity: In a bank transfer, either both the debit and the credit happen, or neither does.
- Consistency: A transaction can't leave an account balance negative if a constraint forbids it.
- Isolation: Two simultaneous seat bookings for the last airline seat won't both succeed.
- Durability: Once a transaction commits, it survives a server crash immediately afterward, because it was written to a durable log.

**2. What does each letter in BASE stand for, and how does it differ from ACID?**
Basically Available, Soft state, Eventual consistency. Unlike ACID, which blocks/delays a response until correctness is guaranteed, BASE always returns a response quickly, tolerates state that changes in the background as replicas catch up, and only guarantees replicas converge *eventually*, not immediately.

**3. How do ACID and BASE relate to the CAP theorem?**
CP-leaning systems (favor consistency over availability during a partition) tend to offer ACID guarantees, since they prioritize correctness even if it means blocking or rejecting requests. AP-leaning systems (favor availability over consistency) tend to offer BASE guarantees, accepting temporary staleness in exchange for always responding.

**4. Name the four SQL standard isolation levels and order them from weakest to strongest.**
Read Uncommitted (weakest, allows dirty reads) → Read Committed → Repeatable Read → Serializable (strongest, transactions behave as if run sequentially). Stronger isolation reduces anomalies but increases locking/contention and reduces throughput.

**5. What is the difference between "Consistency" in ACID and "Consistency" in CAP?**
ACID's Consistency means a transaction moves the database from one valid state to another, respecting schema constraints and rules. CAP's Consistency means all nodes in a distributed system return the same, most recent data at the same time. They are related in spirit but describe different scopes — one is per-transaction data validity, the other is cross-node data agreement.

**6. Would you normalize or denormalize the schema for a read-heavy analytics dashboard, and why?**
Denormalize. Analytics dashboards typically run aggregate queries over large volumes of data and prioritize fast reads over storage efficiency or write simplicity; pre-joining or flattening data (e.g., into wide tables or embedded documents) avoids expensive joins at query time, and the data is often read far more often than it's written.

**7. Give an example of an "update anomaly" that normalization prevents.**
If a customer's address is duplicated across a thousand order rows instead of stored once in a Users table, updating their address means either updating all thousand rows (error-prone) or leaving some rows stale, producing inconsistent data. Normalization stores the address once, so one update fixes it everywhere.

**8. Why do distributed NoSQL databases often favor BASE over ACID?**
Enforcing strict ACID-style consistency and isolation across many geographically distributed nodes requires heavy coordination (e.g., distributed locks or consensus protocols), which adds latency and can reduce availability during network issues. BASE avoids that coordination cost by allowing temporary staleness, which better suits systems that need to scale horizontally and stay available globally.

**9. What is a practical downside of aggressive denormalization?**
Write complexity increases because a single logical update may require updating multiple duplicated copies of the same data. This also introduces a consistency risk — if one copy is missed or delayed, different reads may return different, conflicting versions of the same fact.

**10. Describe 1NF, 2NF, and 3NF at a conceptual level.**
1NF: every column holds a single, atomic value (no lists or repeating groups in one field). 2NF: every non-key column depends on the entire primary key, not just part of a composite key. 3NF: every non-key column depends only on the key, not on other non-key columns (no transitive dependencies).

**11. A social media platform shows a "like count" on posts that occasionally lags by a few seconds across different users. Is this a bug? Explain.**
Not necessarily a bug — it's a natural consequence of choosing an eventually consistent (BASE) architecture to support massive scale and availability. As long as the count converges to the correct value shortly after writes stop, brief staleness on a non-critical, high-volume counter is an acceptable trade-off for speed and uptime.

**12. If you were designing a core banking ledger, would you choose ACID + normalized or BASE + denormalized? Justify your answer.**
ACID + normalized. Financial ledgers require strict correctness — no lost or duplicated transactions, no stale balances that could allow overdrafts or double-spending — and normalization ensures each balance and account fact is stored once, avoiding inconsistency. The cost of extra coordination and slightly higher latency is well worth it given the high cost of an incorrect financial transaction.
