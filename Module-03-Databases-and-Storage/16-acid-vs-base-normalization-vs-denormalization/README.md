# ACID vs BASE, Normalization vs Denormalization

**Difficulty:** Intermediate
**Estimated length:** 12-18 min
**Prerequisites:** [SQL vs NoSQL: Choosing the Right Database](../11-sql-vs-nosql/README.md), [CAP Theorem & PACELC Explained](../15-cap-theorem-and-pacelc/README.md)

## Learning Objectives

By the end of this video, you will be able to:

- Explain each letter of ACID and each word of BASE, with a concrete example for each.
- Connect ACID and BASE back to the CAP theorem and PACELC trade-offs.
- Describe the four standard transaction isolation levels at a conceptual level.
- Explain normalization (1NF/2NF/3NF) and denormalization, and the trade-offs between them.
- Decide, for a given scenario, whether a normalized/ACID design or a denormalized/BASE design is the better fit.

## Script

### Hook/Intro

Hey everyone, welcome back. In the last video we talked about the CAP theorem and PACELC — the idea that when a network partition happens, a distributed system has to choose between staying consistent or staying available, and that even without a partition, there's a trade-off between latency and consistency.

Today we're going to zoom into the consistency side of that trade-off and ask: what does "consistency" actually feel like at the database level? That's where two acronyms come in: ACID and BASE. Here's the connection to what we already know — CP-leaning systems, the ones that favor consistency over availability, tend to give you ACID guarantees. Think traditional relational databases like PostgreSQL or MySQL running in single-node or strongly consistent configurations. AP-leaning systems, the ones that favor availability, tend to give you BASE guarantees instead. Think Cassandra, DynamoDB, or a globally replicated document store. Neither one is "better" — they're just different answers to the same underlying question about how much a system is willing to slow down or say "no" in order to guarantee correctness.

And once we understand that, we'll pivot to a closely related but distinct topic: how you actually shape your data — normalized or denormalized — because that decision is deeply tied to which consistency model you're operating under.

### ACID, in Depth

Let's start with ACID. It stands for Atomicity, Consistency, Isolation, and Durability, and it's the classic guarantee that relational databases give you for transactions.

**Atomicity** means a transaction is all-or-nothing. If you're transferring $100 from Alice's account to Bob's account, that involves two writes: debit Alice, credit Bob. Atomicity guarantees that either both writes happen, or neither does. If the system crashes right after debiting Alice but before crediting Bob, the database rolls back the whole transaction. You never end up with money vanishing into thin air.

**Consistency** — and note this is a different "C" than the one in CAP, which trips a lot of people up — means a transaction takes the database from one valid state to another valid state, according to all defined rules: constraints, foreign keys, triggers, cascades. If your schema says an account balance can never go negative, consistency guarantees that no committed transaction will ever violate that rule.

**Isolation** means concurrent transactions don't interfere with each other. If two people are booking the last seat on a flight at the same second, isolation ensures the database handles them one at a time from a correctness standpoint — you won't sell the same seat twice, even though both requests arrived in parallel.

**Durability** means once a transaction is committed, it survives — even a power failure or crash immediately after. The database typically writes to a transaction log on disk before acknowledging the commit, so that data isn't just sitting in memory waiting to be lost.

Put together, ACID is what lets you trust a database with money, inventory counts, or anything where "approximately correct" isn't good enough.

### A Quick Word on Isolation Levels

Since isolation is really a spectrum and not a single behavior, it's worth knowing the four standard isolation levels defined by the SQL standard, even just at a high level. **Read Uncommitted** is the loosest — you might see other transactions' uncommitted changes, so-called "dirty reads." **Read Committed** — the default in many databases like PostgreSQL — only lets you see data that's been committed, but a repeated read within the same transaction might return different values if another transaction commits in between. **Repeatable Read** fixes that: within a transaction, the same query always returns the same rows. And **Serializable** is the strictest — transactions behave as if they ran one after another, completely isolated, even though they're really running concurrently. The tighter the isolation, the safer you are, but generally the more contention and the slower your throughput. You don't need to memorize the details right now — just know this dial exists and that it's a trade-off knob within ACID itself.

### BASE, in Depth

Now let's flip to BASE, which is the model many distributed and NoSQL systems adopt instead. It stands for Basically Available, Soft state, and Eventual consistency.

**Basically Available** means the system prioritizes responding to every request, even if that response isn't guaranteed to reflect the absolute latest write. Rather than blocking or erroring out during a network hiccup, it degrades gracefully and still gives you an answer.

**Soft state** means the system's state can change over time even without new input, simply because of replication catching up in the background. Data isn't locked down the way it is under ACID's isolation — it's in flux as changes propagate across nodes.

**Eventual consistency** is the big one: if you stop writing to a piece of data, all replicas will eventually converge to the same value — but there's no guarantee about exactly when. You might read slightly stale data from one replica right after a write landed on another.

Contrast that directly with ACID: where ACID says "I will make you wait until I can guarantee correctness," BASE says "I will always answer you, and correctness will catch up shortly." That's exactly the AP side of CAP theorem showing up in database design. Distributed NoSQL systems favor BASE because enforcing strict ACID-style isolation and consistency across many geographically spread nodes is expensive — it requires coordination, and coordination costs latency and availability. If you're building something like a "like" counter or a news feed, a few seconds of staleness is a totally acceptable price for a system that never goes down and always responds fast.

### When to Choose ACID vs BASE

So how do you choose? Ask what happens if the data is wrong or stale, even briefly. For financial ledgers, inventory systems, or anything involving money or unique constraints — like usernames or seat assignments — pick ACID. The cost of a mistake is high, and correctness matters more than raw availability. For things like social media feeds, analytics dashboards, product view counters, or activity logs — pick BASE. The cost of a few seconds of staleness is low, and you'd rather have blazing speed and 24/7 availability across the globe.

### Normalization vs Denormalization

Now let's pivot to a related but distinct decision: how you shape your data.

**Normalization** is the relational database discipline of organizing data to eliminate redundancy. At a conceptual level: **First Normal Form (1NF)** says every column holds a single, atomic value — no comma-separated lists crammed into one field. **Second Normal Form (2NF)** says every non-key column must depend on the whole primary key, not just part of it — this matters for tables with composite keys. **Third Normal Form (3NF)** says every non-key column must depend only on the key, not on other non-key columns. In practice, normalization means splitting data into separate tables — Users, Orders, Products — connected by foreign keys, so each fact is stored exactly once. This avoids "update anomalies": imagine a customer's address stored redundantly in a thousand order rows. If they move, you either update a thousand rows or end up with inconsistent addresses. Normalization fixes that by storing the address once, in the Users table.

**Denormalization** is the deliberate reverse: duplicating data across records to avoid needing joins at read time. If your social media app needs to render a feed instantly, you might store a copy of the author's name and avatar directly inside each post document, rather than joining against a Users collection on every single read. This is extremely common in NoSQL and analytics systems, where read speed at scale matters more than storage efficiency or a single source of truth.

### The Trade-offs

Normalization gives you a single source of truth, less storage, and safer writes — but reads that need to combine related data require joins, which can get expensive at scale. Denormalization gives you fast reads with no joins, which is great for high-traffic, read-heavy workloads — but writes become more complex, because updating one fact might mean touching many duplicated copies, and you introduce a real risk of those copies drifting out of sync with each other. It's the classic space-versus-speed and write-simplicity-versus-read-simplicity trade-off, and — notice this — it maps almost perfectly onto ACID versus BASE. Strongly consistent, normalized schemas tend to go hand-in-hand with ACID transactional databases. Denormalized, duplicated data tends to go hand-in-hand with BASE, eventually consistent systems, because keeping duplicates perfectly in sync across a distributed system in real time is exactly the kind of coordination BASE systems are designed to avoid.

### Real-World Example

Let's ground this. A bank's core ledger system is a textbook ACID and normalized use case. Account balances, transaction history, and customer records live in a normalized relational schema with strict foreign keys, and every transfer runs inside an ACID transaction — atomic, isolated, durable — because a single lost or double-applied transaction is a serious problem, both financially and legally.

Now compare that to a social media feed. When you open the app, you want your feed to load in milliseconds, showing hundreds of millions of users content that's constantly being generated. That system denormalizes aggressively — each post might carry a duplicated snapshot of the author's profile info, like counts are approximate and update asynchronously, and different users might see a slightly different, slightly stale version of the like count for a few seconds. That's BASE in action: basically available, eventually consistent, optimized for read speed and uptime over perfect real-time accuracy.

### Recap

Let's recap. ACID — Atomicity, Consistency, Isolation, Durability — gives you strong, trustworthy transactional guarantees, typically at the cost of availability and latency under distributed conditions. BASE — Basically Available, Soft state, Eventual consistency — trades strict correctness for availability and speed, which is why distributed and NoSQL systems favor it. Separately, normalization organizes data to eliminate redundancy and protect write correctness, while denormalization duplicates data to accelerate reads at the cost of write complexity and consistency risk. And these two decisions tend to travel together: ACID pairs naturally with normalized schemas, and BASE pairs naturally with denormalized ones.

### What's Next

Now that we understand the consistency trade-offs baked into how we store data, next up we're moving into Module 4, starting with "Caching Strategies and Cache Invalidation." Once you put a cache in front of any of these storage systems, you introduce yet another copy of your data — and yet another consistency challenge: how do you know when that cached copy is stale, and what do you do about it? See you there.

## Key Takeaways

- ACID (Atomicity, Consistency, Isolation, Durability) provides strong transactional guarantees and is the natural fit for CP-leaning, single-source-of-truth systems like financial ledgers.
- BASE (Basically Available, Soft state, Eventual consistency) trades strict correctness for availability and speed, and is the natural fit for AP-leaning, distributed NoSQL systems.
- Isolation levels (read uncommitted, read committed, repeatable read, serializable) are a tunable spectrum within ACID itself, trading correctness for concurrency and throughput.
- Normalization reduces redundancy and prevents update anomalies but requires joins; denormalization duplicates data for fast reads but adds write complexity and consistency risk.
- ACID/normalization and BASE/denormalization tend to pair together naturally, both reflecting the same underlying consistency-vs-availability/speed trade-off from CAP and PACELC.
