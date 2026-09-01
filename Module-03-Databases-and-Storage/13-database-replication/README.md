# Database Replication: Master-Slave & Master-Master

**Difficulty:** Intermediate
**Estimated video length:** 12-18 min
**Prerequisites:** [Database Indexing Explained (B-Trees, Hash Indexes)](../12-database-indexing-explained/README.md), [Availability, Reliability, and Fault Tolerance](../../Module-01-Foundations/05-availability-reliability-and-fault-tolerance/README.md)

## Learning Objectives

- Explain what database replication is and why a single database instance is both a reliability risk and a scaling bottleneck.
- Compare synchronous and asynchronous replication and articulate the durability/consistency vs. latency trade-off between them.
- Describe how Master-Slave (leader-follower) replication works, including read scaling, replication lag, and failover.
- Describe how Master-Master (multi-leader) replication works, including write conflict detection and resolution strategies.
- Evaluate which replication topology fits a given scenario based on read/write patterns, consistency needs, and geographic distribution.

## Script

### Hook/Intro

Picture this: you've built an app, it's got one database server, and it's humming along nicely. Then one day, that server's disk fails. Or maybe it doesn't fail — it just can't keep up, because you've gone from a thousand users to a million, and every single read and write is hitting that one machine. Either way, you have a problem, and it's the same root cause in both cases: you have a single point of failure and a single point of scale. One machine going down takes your whole app down with it, and one machine's capacity is a hard ceiling on how much traffic you can serve.

Today we're solving that problem with **database replication** — the practice of copying data from one database node to one or more other nodes, so you have multiple machines holding the same data instead of just one. Replication is one of the most foundational techniques in distributed systems, and it shows up under the hood of nearly every production database you've ever used — PostgreSQL, MySQL, MongoDB, all of them support it natively. We're going to cover what replication actually is, the difference between synchronous and asynchronous replication, and then dig into the two big topologies you'll be asked about in any system design interview: Master-Slave and Master-Master.

### What Replication Actually Is

At its core, replication just means: keep more than one copy of your data, on more than one machine, and keep those copies in sync. Why bother? Two big reasons. First, **availability and durability** — if one node dies, another node already has the data and can take over, so you don't lose data and you don't go down. Second, **read scaling** — if you have multiple copies of the data sitting on multiple machines, you can spread read traffic across all of them instead of hammering a single server.

But replication isn't free. The moment you have multiple copies of the same data, you have to answer a hard question: how do you keep them in sync, and what happens when they temporarily disagree? That's where synchronous versus asynchronous replication comes in.

### Synchronous vs. Asynchronous Replication

With **synchronous replication**, when a client writes data, the primary node sends that write to the replica, and it waits for the replica to confirm it has received and applied the write before telling the client "your write succeeded." Think of it like sending a certified letter — you don't consider the job done until you have the signed receipt back. The upside is strong durability and consistency: if the primary dies the instant after acknowledging the write, you know the replica already has that data too, so nothing is lost. The downside is latency — every write now has to wait on a network round trip to the replica, which is slow, and if the replica is unreachable, your writes can stall entirely.

With **asynchronous replication**, the primary acknowledges the write to the client immediately, and then sends the update to the replicas in the background, whenever it gets to it. This is like dropping a regular letter in the mailbox — you move on with your day without waiting for a receipt. Writes are fast because you're not waiting on the network round trip to a replica. But now there's a window where the primary has data the replicas don't have yet. If the primary crashes during that window, that data can be lost. This gap between "the primary has it" and "the replica has it" is called **replication lag**, and it's one of the most important concepts in this whole topic — we'll come back to it constantly.

Most real-world systems default to asynchronous replication for performance, and use synchronous replication selectively — for example, requiring just one replica to synchronously acknowledge critical writes, while the rest replicate asynchronously. That's a trade-off you tune based on how much durability you need versus how much latency you can tolerate.

### Master-Slave (Leader-Follower) Replication

The most common replication topology is **Master-Slave**, also increasingly called **leader-follower** replication, since "master/slave" terminology is being phased out across the industry in favor of "primary/replica" or "leader/follower" — but you'll still see master-slave in a lot of documentation and interview questions, so it's worth knowing both names.

Here's how it works: you designate one node as the **master** (or leader), and all the other nodes are **slaves** (or followers, or read replicas). All writes — inserts, updates, deletes — go exclusively to the master. The master then streams those changes to each follower, usually asynchronously. Reads, on the other hand, can go to *any* node — the master or any of the followers. This is the big win: since most applications are read-heavy, you can scale your read capacity almost linearly just by adding more follower replicas, all without touching your write capacity or your data model.

But this introduces our friend replication lag. Because replication to followers is typically asynchronous, there's a small delay — often milliseconds, but sometimes seconds under heavy load — between when a write lands on the master and when it's visible on a follower. This creates a classic and very real bug called the **read-your-writes problem**: a user posts a comment, the write goes to the master, the app immediately redirects them to a page that reads from a follower, and the follower hasn't received that write yet — so the user sees their own comment missing. Classic fixes include routing a user's own reads to the master for a short window after they write, tracking a "read your own writes" token tied to replication position, or simply routing session-critical reads to the master.

Now, what happens when the master itself dies? This is called **failover**, and it's the trickiest operational part of master-slave replication. The system — either an automated tool or a human operator — has to detect that the master is down, pick one of the followers to promote to be the new master, and redirect all future writes to it. This takes time, during which writes are typically unavailable, and there's a risk of losing whatever writes hadn't yet replicated to the promoted follower if you were using asynchronous replication. Tools like PostgreSQL's Patroni or MySQL Group Replication automate much of this, but it's never instantaneous or risk-free.

### Master-Master (Multi-Leader) Replication

**Master-Master**, or **multi-leader replication**, takes a different approach: instead of one master accepting all writes, *multiple* nodes can each accept writes directly, and they replicate changes to each other. This solves a problem master-slave can't: write scaling and write availability across multiple locations. If you have users in the US and Europe, you can run a master in each region, and users write to whichever master is geographically closest to them — much lower write latency, and if one region's master goes down, the other keeps accepting writes without any failover process at all.

But multi-leader replication opens up a genuinely hard problem: **write conflicts**. If a user in the US updates a record at the same moment a user in Europe updates the *same* record, both masters accept the write locally, and then when they replicate to each other, you have two different values claiming to be correct for the same piece of data. Something has to decide who wins.

The simplest strategy is **last-write-wins (LWW)**: attach a timestamp to every write, and when a conflict is detected, whichever write has the later timestamp survives. It's simple and it's what many systems use by default, but it's also a blunt instrument — it can silently discard a legitimate write just because clocks are slightly out of sync, or because two writes happened close together. More sophisticated systems use **vector clocks** — a data structure that tracks the causal history of updates across nodes, so the system can tell whether one write actually *happened after* another (and should overwrite it), or whether they happened truly concurrently and need to be flagged for merging, sometimes even surfaced back to the application or the user to resolve. Some databases, like CouchDB, expose conflicts to the application layer directly rather than silently picking a winner.

Master-master shines in **multi-region active-active** architectures, where you genuinely need every region to accept both reads and writes for low latency and regional fault tolerance — think globally distributed collaboration tools or shopping carts. But you pay for that flexibility with real operational and application complexity around conflict resolution.

### Comparing the Two Topologies

Let's line these up directly. In terms of **write scaling**, master-slave is limited — all writes go through one node — while master-master lets you scale writes across multiple nodes and regions. For **read scaling**, both handle it well, since both let you add followers or additional leaders to absorb read traffic. For **conflict risk**, master-slave has essentially none, since there's only ever one writer; master-master has real conflict risk that you must actively design for. For **complexity**, master-slave is simpler to reason about and operate; master-master is meaningfully more complex due to conflict resolution and multi-directional replication. For **failover**, master-slave requires an explicit promotion process with some downtime or data-loss risk; master-master has no single point of failure for writes, since other masters keep accepting writes if one goes down. The rule of thumb: reach for master-slave by default, since it's simpler and covers the vast majority of read-heavy workloads, and only reach for master-master when you have a specific need for multi-region write availability that justifies the added complexity.

### Real-World Example

Consider a read-heavy blogging platform: for every one write (a new post or comment), there might be a thousand reads (people browsing and viewing content). A textbook master-slave setup handles this beautifully — one master handles the relatively rare writes, and you place, say, five read replicas behind a load balancer to absorb the massive read volume, scaling read capacity by simply adding more replicas as traffic grows. Now contrast that with a global collaborative document editor with active users in North America, Europe, and Asia, all needing fast writes with low latency and continued availability even if one region has an outage. That's a textbook master-master case: a master per region, asynchronous cross-region replication, and a conflict resolution strategy — often vector clocks or operational-transform-style merging — to reconcile concurrent edits.

### Recap

Let's recap. Replication means keeping multiple copies of your data across multiple nodes for availability and read scaling. Synchronous replication trades latency for durability by waiting for replica confirmation; asynchronous replication trades a small durability risk for speed, and creates replication lag. Master-slave replication routes all writes to one master and distributes reads across followers — simple, but limited in write scaling and requires a failover process when the master dies. Master-master replication lets multiple nodes accept writes, scaling write throughput and enabling multi-region active-active setups, at the cost of needing a conflict resolution strategy like last-write-wins or vector clocks.

### What's Next

Replication is fantastic for scaling *reads* — you can throw as many replicas as you want at read traffic. But notice what it does *not* solve: it doesn't help you scale *writes* past what one machine (or, in master-master, a handful of machines each holding the full dataset) can handle, and it doesn't help when your total dataset is simply too large to fit on one machine's disk at all. For that, you need a fundamentally different technique: splitting your data itself across nodes, rather than copying all of it everywhere. That's exactly what we're covering in the next video: **Database Sharding & Partitioning Strategies**. See you there.

## Key Takeaways

- A single database instance is both a single point of failure and a hard ceiling on scale — replication solves both by maintaining multiple copies of data across nodes.
- Synchronous replication favors durability/consistency at the cost of write latency; asynchronous replication favors speed at the cost of a replication lag window where data could be lost on primary failure.
- Master-Slave (leader-follower) replication sends all writes to one master and distributes reads across followers, giving strong read scaling but requiring a failover process and being subject to the read-your-writes problem caused by replication lag.
- Master-Master (multi-leader) replication accepts writes on multiple nodes, enabling write scaling and multi-region active-active architectures, but requires a conflict resolution strategy such as last-write-wins or vector clocks.
- Replication scales reads (and, with master-master, writes to a degree) but does not solve total dataset size or fundamental write-throughput limits — that's the job of sharding, covered next.
