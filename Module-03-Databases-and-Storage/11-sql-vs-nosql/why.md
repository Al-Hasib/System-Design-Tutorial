# Why This Topic Matters: SQL vs NoSQL

> **In one sentence:** The database is the hardest thing in your architecture to change later, and picking one based on hype rather than access patterns is the most expensive mistake available to you at design time.

## The World Before This Idea

For about thirty years there was no choice to make — you used a relational database, because that's what existed. Then a wave of systems built for internet scale (Dynamo, BigTable, Cassandra, MongoDB, Redis) arrived, along with a marketing narrative that relational databases "don't scale."

That narrative caused enormous damage. Teams migrated transactional, highly relational workloads onto document stores, then spent years reimplementing joins in application code, discovering they'd lost transactions they actually needed, and eventually migrating back. Meanwhile other teams jammed a genuinely key-value workload into Postgres and wondered why it fell over.

Both failures have the same root: choosing a database by category instead of by the shape of the data and the queries.

## The Problems It Solves

### 1. Joins reimplemented badly in application code
**What you see:** A "get order with customer and items" endpoint issues 40 separate queries, each returning one document, stitched together in a loop in the application.

**Why it happens:** Document stores deliberately don't join. If your data is genuinely relational — entities referencing each other in many directions — you still need the joins; you've just moved them into your own code, where they're slower, unindexed, and untested.

**How this topic solves it:** It teaches you to look at the *access pattern* first. Highly interconnected data queried in many combinations is what relational databases and their query planners are built for. That's not legacy; that's fit.

### 2. Data integrity lost without anyone deciding to lose it
**What you see:** Orders referencing deleted products. A user record with `status: "activ"`. Two documents that disagree about the same fact.

**Why it happens:** Schemas, foreign keys, and constraints are enforcement, not bureaucracy. Remove them and every writer in the system — including the buggy one, and the one-off script someone ran at 2 a.m. — becomes responsible for correctness.

**How this topic solves it:** Schema-on-write versus schema-on-read is a deliberate trade, not a free upgrade. "Flexible schema" means "the schema now lives in your application code, in every application that touches the data, forever." Sometimes that's the right call. It should be a call.

### 3. A single node that can't take the write volume
**What you see:** You're ingesting a million events a second. One primary node cannot absorb it, and every shard you bolt onto the relational database adds operational pain.

**Why it happens:** Classic relational databases were architected around a single writable node with strong guarantees. Horizontal write scaling is possible but is a retrofit.

**How NoSQL solves it:** Systems like Cassandra and DynamoDB were designed from the ground up around partitioning — data is spread by key across nodes, and adding nodes adds write throughput close to linearly. For write-heavy, key-addressable workloads (time series, events, sensor data, sessions), this is a genuine architectural advantage rather than marketing.

### 4. Paying for guarantees you don't need
**What you see:** A session store or a cache of computed results sits in your main relational database, competing for connections with business-critical transactions and dragging down the whole system.

**Why it happens:** Defaulting to one database for everything.

**How this topic solves it:** Polyglot persistence — the recognition that different data has different needs. Sessions in Redis, documents in Mongo, transactions in Postgres, search in Elasticsearch, events in Kafka. The cost is more systems to operate, so the answer isn't "always split" — but "always one database" is equally unthinking.

## The Price You Pay

- **Every extra datastore is an operational commitment.** Backups, monitoring, upgrades, expertise, on-call runbooks. Two databases is meaningfully more than twice the work of one.
- **NoSQL demands you know queries up front.** Denormalized, query-first models are fast for the patterns you designed for and near-impossible for the ones you didn't. Requirements change; data models resist.
- **Migrations between families are brutal.** Changing from Postgres to DynamoDB isn't a driver swap; it's a redesign of the data model and often the application.
- **Modern databases blur the line.** Postgres has strong JSON support, MongoDB has transactions, DynamoDB has secondary indexes. The clean dichotomy is less clean than the debate suggests, which is another reason to reason from workload rather than category.

## When You Need It — and When You Don't

| Reach for relational when | Reach for NoSQL when |
|---|---|
| Data is highly relational and queried many ways | Access is primarily by a known key |
| You need multi-row ACID transactions | You need very high write throughput and horizontal scale |
| Query patterns will evolve unpredictably | Query patterns are few, known, and stable |
| Integrity constraints matter (money, inventory, identity) | The shape of records genuinely varies per record |
| Scale is "large" but not "internet-scale" | Availability under partition beats strict consistency |

**The honest default:** start relational unless you have a specific, articulated reason not to. Postgres handles far more load than people assume, and "we might need scale someday" is not a reason.

## Why This Shows Up in Interviews

"Which database would you use and why?" is asked in nearly every design round, and the trap is answering with a product name. The expected answer derives the choice: here's the data shape, here's the read/write ratio, here's the consistency requirement, therefore this family, therefore this product. A candidate who says "DynamoDB, because our access pattern is a single-key lookup by user ID at very high volume and we don't need cross-entity transactions" has answered well. "MongoDB, it's more scalable" has not.

## How It Connects

This choice drives much of what follows. **Indexing** determines read performance within whichever you choose. **Replication** and **sharding** are how either family scales. **CAP/PACELC** explains the consistency-availability trade-offs NoSQL systems make explicit. **ACID vs BASE** and **normalization vs denormalization** are the modeling consequences of this decision. **LSM trees vs B-trees** explains why these families have such different write performance at the storage-engine level.

**Next:** [Database Indexing Explained](../12-database-indexing-explained/why.md) — how either kind of database finds your data fast.
