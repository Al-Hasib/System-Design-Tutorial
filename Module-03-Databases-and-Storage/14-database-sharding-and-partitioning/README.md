# Database Sharding & Partitioning Strategies

**Difficulty:** Intermediate/Advanced
**Estimated length:** 12-18 min
**Prerequisites:**
- [Database Replication: Master-Slave & Master-Master](../13-database-replication/README.md)
- [Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md)

## Learning Objectives

By the end of this video, you should be able to:

- Explain what sharding and partitioning mean, and how they differ from replication.
- Distinguish vertical (functional) partitioning from horizontal partitioning (sharding).
- Compare range-based, hash-based, and directory-based sharding strategies, including their trade-offs.
- Evaluate what makes a good shard key and avoid common hotspot pitfalls.
- Identify the operational challenges sharding introduces, like cross-shard joins and rebalancing.

## Script

### Hook/Intro

In the last video, we talked about replication — copying your data across multiple nodes so you can survive a failure and spread out your read traffic. That solves a real problem. If your app is read-heavy, replication lets you add read replicas and just keep going. But here's the thing replication does *not* solve: write scaling, and raw dataset size.

Think about it. In a master-slave setup, every single write still has to go through one primary node. It doesn't matter if you have ten read replicas or a hundred — writes are still bottlenecked on that one machine's CPU, memory, and disk I/O. And even if you only ever read, at some point your dataset itself just gets too big to fit on one machine, or too big to query efficiently even if it technically fits. Replication gives every node a *full copy* of the same data. That's great for availability, but it does nothing to shrink the amount of data each node has to manage.

So what do you do when one database server simply cannot hold, or cannot handle, all your data and all your writes? You split it up. That's sharding — and it's the subject of today's video.

### What Is Sharding / Partitioning?

Let's define terms, because people use "partitioning" and "sharding" a little loosely.

Partitioning, broadly, means dividing your data into smaller, more manageable pieces. Sharding is a specific *kind* of partitioning: splitting data across multiple separate database instances — different servers, potentially in different racks or regions — where each instance, called a shard, holds only a subset of the total data.

Here's an analogy. Imagine a massive library with ten million books, all crammed into one building. Every visitor, every librarian, every delivery truck funnels through that one building's one entrance. Eventually the building is full, and the hallways are jammed. Sharding is like opening ten branch libraries instead, each holding one-tenth of the books. Now, ten front doors handle traffic instead of one, and each branch only needs to store and index its own slice of the collection.

### Vertical vs Horizontal Partitioning

Before we go further, let's separate two concepts that both get called "partitioning."

**Vertical partitioning** splits your data by *columns* or by *feature/table*. For example, you might put your `users` table on one database and your `orders` table on a completely different database, because they're accessed by different services. Or within a single table, you might split rarely-used, large columns — like a user's biography or profile picture blob — into a separate table so that the "hot" columns stay small and fast to scan. This is really just functional decomposition of your schema across different data stores.

**Horizontal partitioning**, which is what we call sharding, splits data by *rows*. You take a single logical table — say, `users` — and you split its rows across multiple database instances. Shard 1 might hold users 1 through 10 million, shard 2 holds the next 10 million, and so on. Every shard has the identical schema; they just hold different rows. This is the technique that actually lets you scale both storage and write throughput horizontally, because each shard is an independent database that only deals with its own slice of rows.

Today we're focused mostly on horizontal partitioning — sharding — because that's the piece that unlocks true horizontal scale for both reads and writes.

### Sharding Strategies

So once you've decided to split rows across shards, how do you decide *which row goes to which shard*? There are three classic strategies.

**Range-based sharding.** You pick a shard key — say, `user_id` — and you assign contiguous ranges of that key to each shard. Users 1 to 1,000,000 go to shard A, 1,000,001 to 2,000,000 go to shard B, and so on. The big advantage here is that range queries are cheap and natural — "give me all users between ID 500,000 and 600,000" hits exactly one shard. The problem is hotspots. If your key is something like a signup timestamp or an auto-incrementing ID, all your *newest* — and often most active — data lands on the *last* shard, while your older shards sit relatively idle. You get uneven load even though you have "even" data distribution by count.

**Hash-based sharding.** Here, you run the shard key through a hash function, and the hash result — typically modulo the number of shards — determines which shard the row lives on. Because a good hash function scatters keys pseudo-randomly, this tends to spread both data volume and load very evenly across shards — no more hotspots concentrated in one range. The trade-off is that you lose the ability to do efficient range queries. "Give me all orders between March and April" no longer maps to one shard; you might have to query every shard and merge results, because rows that were sequential in the original key are now scattered randomly.

**Directory-based sharding.** Instead of computing where data lives via a formula, you maintain an explicit lookup service — a directory — that maps each key, or each range of keys, to a specific shard. Need to know where user 42 lives? Ask the directory service, it tells you "shard 3." This is by far the most *flexible* approach: you can move individual keys between shards, rebalance unevenly loaded shards, and add capacity without needing everything to follow a strict mathematical formula. The cost is added complexity and an extra network hop on every single query — plus the directory service itself becomes a critical piece of infrastructure that needs to be fast, available, and consistent.

Now, one problem shared by both range and plain hash-modulo sharding is resharding pain. If you add or remove a shard with naive hash-modulo — hash mod N — changing N reshuffles almost every key's target shard, meaning you'd have to move nearly all your data. There's a technique called **consistent hashing** that dramatically reduces how much data needs to move when you scale the shard count up or down. I'm not going to go deep on it here — it deserves its own focused explanation with diagrams — but just know it exists, it's widely used in real distributed systems, and we'll cover it in detail in a later video.

### Choosing a Shard Key

Picking the right shard key is arguably the single most consequential decision in a sharding design, because it's painful to change later. A few things to look for:

First, **cardinality** — you want a key with enough distinct values that data actually spreads across all your shards. Sharding by `country` might sound reasonable, but if 80% of your users are in one country, that shard becomes a hotspot no matter how you slice it.

Second, **access patterns**. Look at your most common queries. If almost every query filters by `tenant_id` or `customer_id`, sharding on that key means most queries can be routed to a single shard without needing to fan out and merge results across all of them — that's a huge performance and simplicity win.

Third, **avoiding hotspots**. Watch out for keys correlated with time or sequential IDs, celebrity/power-user effects where one key gets vastly more traffic than others, and skewed real-world distributions in general.

### Challenges of Sharding

Sharding isn't free — it introduces real operational complexity.

**Cross-shard joins and transactions** get hard. If related data lives on different shards, a join now means querying multiple databases and combining results in application code, and a transaction spanning two shards needs distributed transaction coordination — think two-phase commit or saga patterns — instead of a simple local ACID transaction.

**Rebalancing** is painful. As data grows unevenly, you'll eventually need to move data between shards or add new shards, and depending on your strategy, that can mean migrating huge volumes of data with minimal downtime — a genuinely hard operational problem.

And overall **operational complexity** goes up substantially. Instead of managing one database, you're managing N databases, each needing monitoring, backups, schema migrations applied consistently, and capacity planning — multiplied by however many shards you run.

### Real-World Example

Let's make this concrete. Say you're building an e-commerce platform, and your `orders` table has grown past what a single Postgres instance can comfortably handle.

One approach: shard by `customer_id`, using hash-based sharding across, say, 16 shards. Every order a customer places hashes to the same shard, so "get all orders for this customer" — one of your most common queries — always hits exactly one shard. New customers and their order volume distribute evenly because the hash function doesn't care about signup date or region.

Alternatively, imagine a global users table sharded by `user_id` using a hash. When a user logs in, your application hashes their ID, looks up which of your shards owns that hash range, and routes the query straight there. Simple point lookups by user ID stay fast and single-shard; it's only cross-user analytical queries — like "how many users signed up last week across the whole platform" — that now require fanning out to every shard and aggregating in your application layer.

### Recap

Let's tie it together. Replication copies the same data everywhere to help with availability and read scaling. Sharding — horizontal partitioning — splits *different* rows across different database instances to scale writes and total storage. Vertical partitioning, by contrast, splits your schema by table or column, not by row. For strategy, range-based sharding gives you cheap range queries but risks hotspots; hash-based sharding gives you even distribution but sacrifices range queries; directory-based sharding gives you maximum flexibility at the cost of an extra lookup hop and added infrastructure. Consistent hashing helps ease the resharding pain that comes with any of these. And your shard key choice — driven by cardinality, access patterns, and hotspot avoidance — is the decision that will make or break the whole design.

### What's Next

So now we've covered two of the biggest tools in the distributed-data toolbox: replication for copies, and sharding for splits. Combine them — and you get systems that are both horizontally scaled *and* fault-tolerant. But that combination doesn't come for free. Every time you replicate or shard, you're implicitly making trade-offs between consistency, availability, and latency. In the next video, we're going to make those trade-offs explicit with the CAP theorem and its more nuanced cousin, PACELC — so you understand exactly what you're giving up, and what you're gaining, every time you distribute your data.

## Key Takeaways

- Sharding (horizontal partitioning) splits rows of a table across multiple database instances to scale writes and total storage — something replication alone cannot do.
- Vertical partitioning splits data by table/column; horizontal partitioning (sharding) splits data by row across separate instances.
- Range-based sharding supports efficient range queries but risks hotspots on sequential or time-correlated keys.
- Hash-based sharding distributes load evenly but makes range queries expensive, since they require fanning out to every shard.
- Directory-based sharding offers the most flexibility via a lookup service, at the cost of an extra network hop and added infrastructure.
- Consistent hashing reduces the amount of data movement needed when adding/removing shards (covered in depth later).
- A good shard key has high cardinality, matches your dominant access patterns, and avoids concentrating load on a small number of values.
- Sharding introduces real operational costs: cross-shard joins/transactions, rebalancing, and multiplied operational overhead.
