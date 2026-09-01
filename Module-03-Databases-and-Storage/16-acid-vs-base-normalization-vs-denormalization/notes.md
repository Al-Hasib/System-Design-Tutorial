# Study Notes: ACID vs BASE, Normalization vs Denormalization

## ACID Definitions

| Letter | Term | Definition |
|---|---|---|
| A | **Atomicity** | A transaction is all-or-nothing; partial writes are rolled back. |
| C | **Consistency** | A transaction moves the database from one valid state to another, respecting all constraints, cascades, and triggers. |
| I | **Isolation** | Concurrent transactions do not see each other's uncommitted intermediate states. |
| D | **Durability** | Once committed, data survives crashes and power loss (typically via a write-ahead log). |

> Note: "Consistency" in ACID (per-transaction data validity) is a different concept from "Consistency" in CAP (all nodes see the same data at the same time). Don't conflate them.

## BASE Definitions

| Word | Term | Definition |
|---|---|---|
| BA | **Basically Available** | The system guarantees a response to every request, even under partial failure, rather than blocking. |
| S | **Soft state** | System state may change over time without new input, due to background replication/convergence. |
| E | **Eventual consistency** | If writes stop, all replicas will eventually converge to the same value, with no fixed bound on when. |

## Isolation Levels (SQL Standard, weakest to strongest)

1. **Read Uncommitted** — dirty reads possible (can see uncommitted data from other transactions).
2. **Read Committed** — no dirty reads; default for many databases (e.g., PostgreSQL, Oracle). Non-repeatable reads possible.
3. **Repeatable Read** — same query returns the same rows within a transaction; phantom rows still possible in some implementations. Default in MySQL/InnoDB.
4. **Serializable** — strictest; transactions behave as if executed sequentially. Highest safety, lowest concurrency/throughput.

General rule: stricter isolation = fewer anomalies but more locking/contention and lower throughput.

## ACID vs BASE Comparison

| Aspect | ACID | BASE |
|---|---|---|
| Guarantee | Strong transactional correctness | High availability, approximate correctness |
| Consistency model | Strong / immediate consistency | Eventual consistency |
| CAP alignment | CP-leaning (consistency over availability) | AP-leaning (availability over consistency) |
| Typical systems | PostgreSQL, MySQL, Oracle, SQL Server | Cassandra, DynamoDB, Riak, MongoDB (default configs), CouchDB |
| Coordination cost | Higher (locks, consensus, logs) | Lower (async replication) |
| Use cases | Banking, ledgers, inventory, order processing, anything with uniqueness/money constraints | Social feeds, view/like counters, activity logs, caching layers, IoT telemetry |

## Normalization vs Denormalization

- **Normalization**: organize data into related tables to eliminate redundancy.
  - **1NF**: atomic column values (no repeating groups/lists in a single field).
  - **2NF**: every non-key column depends on the *whole* primary key (relevant for composite keys).
  - **3NF**: every non-key column depends *only* on the key, not on other non-key columns (removes transitive dependencies).
  - Goal: avoid update/insert/delete anomalies (e.g., editing a duplicated address in a thousand rows).
- **Denormalization**: intentionally duplicate data to avoid joins and speed up reads. Common in NoSQL document models (embed related data) and analytics/OLAP systems (wide, flattened tables).

| Aspect | Normalization | Denormalization |
|---|---|---|
| Redundancy | Minimal (single source of truth) | High (data duplicated across records) |
| Read speed | Slower for multi-entity reads (requires joins) | Faster (data embedded together, no joins) |
| Write speed | Faster/simpler (write once) | Slower/more complex (must update all copies) |
| Storage | Lower | Higher |
| Consistency risk | Low (one place to update) | Higher (copies can drift out of sync) |
| Typical use case | OLTP systems, financial/transactional data, relational schemas | Read-heavy feeds, dashboards, analytics, document databases, caching |

## Key Facts

- ACID is defined per-transaction; the isolation spectrum (read uncommitted → serializable) is a knob *within* ACID, not a separate model.
- BASE was coined as an informal counterpoint to ACID, popularized in the context of large-scale distributed/NoSQL systems (e.g., Amazon Dynamo, Google Bigtable-era systems).
- CP-leaning systems (from CAP theorem) tend to offer ACID; AP-leaning systems tend to offer BASE — the mapping isn't a law, but a strong tendency.
- Normalized schemas and ACID transactions tend to travel together; denormalized schemas and BASE/eventual consistency tend to travel together.

## Bullet Summary

- ACID = Atomicity, Consistency, Isolation, Durability — strong guarantees, CP-leaning, relational databases.
- BASE = Basically Available, Soft state, Eventual consistency — weaker guarantees, AP-leaning, distributed/NoSQL databases.
- Isolation levels tune the trade-off between correctness and concurrency inside ACID.
- Normalization reduces redundancy and protects write integrity; denormalization duplicates data to accelerate reads.
- Choose based on the cost of being wrong/stale: high cost → ACID + normalized; low cost, need for scale/speed → BASE + denormalized.
