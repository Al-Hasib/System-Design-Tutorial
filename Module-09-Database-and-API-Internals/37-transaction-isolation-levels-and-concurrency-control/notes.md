# Study Notes: Transaction Isolation Levels & Concurrency Control

## Definitions

- **Dirty read:** Reading another transaction's uncommitted (and possibly-to-be-rolled-back) changes.
- **Non-repeatable read:** Reading the same row twice within one transaction and getting different values, because another transaction committed a change in between.
- **Phantom read:** Re-running the same query and getting a different set of rows, because another transaction inserted/deleted matching rows in between.
- **Pessimistic locking:** Acquiring a lock on data before touching it, forcing other transactions to wait.
- **Optimistic Concurrency Control (OCC):** Proceeding without locks, checking for conflicts only at commit time (typically via a version/timestamp comparison), and retrying on conflict.
- **MVCC (Multi-Version Concurrency Control):** Keeping multiple versions of each row so readers see a consistent snapshot without blocking concurrent writers.
- **Deadlock:** Two (or more) transactions each hold a lock the other needs, and neither can proceed; the database detects this and aborts one.

## The Four SQL Isolation Levels

| Level | Dirty reads | Non-repeatable reads | Phantom reads | Notes |
|---|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible | Weakest; rarely used |
| Read Committed | Prevented | Possible | Possible | Default in PostgreSQL, SQL Server, Oracle |
| Repeatable Read | Prevented | Prevented | Possible (per spec; some engines prevent it anyway) | Default in MySQL/InnoDB |
| Serializable | Prevented | Prevented | Prevented | Strongest; highest coordination cost, most aborts/retries |

## Locking vs. Optimistic Concurrency Control

| | Pessimistic Locking | Optimistic Concurrency Control |
|---|---|---|
| Assumption | Conflicts are likely | Conflicts are rare |
| Mechanism | Acquire lock before acting | Act freely, check version/timestamp at commit |
| Cost when conflicts are rare | Wasted waiting/contention | Minimal — cheap check, rare retries |
| Cost when conflicts are common | Necessary — prevents wasted work | Expensive — frequent aborts and retries |
| Risk | Deadlock (needs detection/resolution) | Requires application-level retry logic |
| Good fit | Hot, high-contention rows (e.g., inventory count) | Most rows, most of the time, in typical apps |

## MVCC in Practice

- Each row can have multiple versions, tagged by the transaction that created them.
- A reader sees the version consistent with its own transaction/statement start time — never blocks on a concurrent writer.
- Writers still must coordinate with other writers on the same row (can't have two conflicting "winners").
- Old versions need cleanup once no longer needed by any active transaction (e.g., PostgreSQL's `VACUUM`); neglecting this causes table bloat.
- Even under MVCC, hot/high-stakes rows may still need explicit pessimistic locking (e.g., `SELECT ... FOR UPDATE`) where a race genuinely cannot be tolerated.

## Key Numbers / Facts

- PostgreSQL's default isolation level is Read Committed; MySQL/InnoDB's default is Repeatable Read.
- `SELECT ... FOR UPDATE` is the standard SQL way to force pessimistic row-level locking within an MVCC database.
- Serializable isolation in PostgreSQL is implemented via Serializable Snapshot Isolation (SSI), which detects conflicts and aborts transactions rather than blocking them outright.

## Summary

- Concurrency without protection produces specific, well-understood anomalies: dirty reads, non-repeatable reads, phantom reads.
- Isolation levels trade off which anomalies are prevented against throughput/coordination cost — pick deliberately, don't leave it at a framework default unexamined.
- Pessimistic locking and optimistic concurrency control are the two engineering strategies for enforcing isolation; MVCC is how most real databases implement this efficiently by avoiding reader/writer blocking through row versioning.
