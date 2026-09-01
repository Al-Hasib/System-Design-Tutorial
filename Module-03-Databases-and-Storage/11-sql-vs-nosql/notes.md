# Notes: SQL vs NoSQL

## Definitions

- **SQL (relational) database** — stores data in fixed-schema **tables** made of rows and columns, with relationships between tables expressed via **foreign keys** and combined at query time via **JOINs**. Queried with SQL (Structured Query Language). Examples: PostgreSQL, MySQL, Oracle, SQL Server.
- **NoSQL database** — "not only SQL"; an umbrella term for non-relational databases with flexible or no fixed schema, generally designed to scale horizontally across many machines. Not one model — four major categories (document, key-value, wide-column, graph).

## SQL vs NoSQL Comparison

| Dimension | SQL (Relational) | NoSQL |
|---|---|---|
| Schema | Fixed, enforced upfront; changes need migrations | Flexible/dynamic; fields can vary per record |
| Scaling | Primarily vertical (bigger machine); harder to shard | Primarily horizontal (more machines); built for sharding |
| Consistency | Strong consistency by default (ACID) | Often eventual consistency (BASE); some offer tunable consistency |
| Query language / flexibility | SQL; supports arbitrary ad-hoc queries and JOINs | Varies by store; typically simpler queries, no/limited joins, data denormalized for known access patterns |
| Data relationships | First-class via foreign keys + JOINs | Usually denormalized/embedded, or (for graph DBs) modeled as first-class nodes/edges |
| Best fit | Structured, relational data; transactions where integrity matters (banking, orders, inventory) | High-scale, fast-changing, or loosely structured data; caching, catalogs, social graphs, event streams |
| Examples | PostgreSQL, MySQL, Oracle, SQL Server | MongoDB, Redis, DynamoDB, Cassandra, Neo4j |

## NoSQL Categories

| Category | Examples | Typical Use Case |
|---|---|---|
| Document | MongoDB, Couchbase | Nested, self-contained objects fetched as a whole — user profiles, product catalogs, CMS content |
| Key-value | Redis, DynamoDB, Memcached | Simple, ultra-fast lookups — caching, sessions, shopping carts |
| Wide-column | Cassandra, HBase | Massive write volume, time-series/event data, range scans across clusters |
| Graph | Neo4j, Amazon Neptune | Relationship-heavy data — social graphs, recommendation engines, fraud detection |

## Key Facts / Concepts

- **ACID** (SQL default): Atomicity, Consistency, Isolation, Durability — transactions fully succeed or fully fail, and committed data survives crashes.
- **BASE** (common in distributed NoSQL): Basically Available, Soft state, Eventually consistent — favors availability and partition tolerance over immediate consistency.
- **Normalization** (SQL norm): store each fact once, reference it via foreign keys — avoids duplication, costs JOIN overhead.
- **Denormalization** (NoSQL norm): duplicate data shaped for how it's read — faster reads for known access patterns, more write-time/storage overhead to keep copies in sync.
- **Polyglot persistence**: using multiple database types in one system, each matched to the sub-problem it fits best (e.g., relational for orders/payments, key-value for sessions/cache, graph for social connections, wide-column for event telemetry).
- Ties directly to the **CAP theorem** (consistency, availability, partition tolerance trade-offs) and horizontal vs. vertical **scaling**, both covered elsewhere in this course.

## Quick Summary

- SQL = structured, relational, strongly consistent, vertically-scaled-by-default, flexible ad-hoc querying.
- NoSQL = flexible schema, horizontally-scaled-by-default, often eventually consistent, optimized for known access patterns.
- Four NoSQL flavors: document, key-value, wide-column, graph — pick based on data shape, not habit.
- Choice depends on: data shape, scale needs, consistency needs, team familiarity/known query patterns.
- Most large real systems use several database types together (polyglot persistence), not just one.
