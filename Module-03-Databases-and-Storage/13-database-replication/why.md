# Why This Topic Matters: Database Replication

> **In one sentence:** A single database holds every byte your business owns on one machine that will eventually fail — replication is how you survive that, and as a bonus, how you scale reads.

## The World Before This Idea

One database server. It has your users, your orders, your entire product. Ask yourself two questions:

1. What happens when that disk fails at 3 a.m.? Best case, you restore last night's backup and lose a day of data while the business is offline for hours.
2. What happens when read traffic grows 10x? The one machine that serves every read also serves every write. Reads starve writes, writes block reads, and both get slower.

Backups alone don't solve either. A backup is a point-in-time copy; restoring it is slow and lossy. Replication is a *continuously updated* copy that's already running.

## The Problems It Solves

### 1. Total data loss from one hardware failure
**What you see:** A disk dies. The most recent backup is 14 hours old. Fourteen hours of orders are gone and cannot be reconstructed.

**Why it happens:** All data exists in exactly one place.

**How replication solves it:** A replica continuously applies the primary's changes, so a second (or third) physically separate copy is always seconds behind. Losing the primary means losing seconds, not hours — and the replica can be promoted to serve traffic rather than restored from tape.

### 2. Read load that crushes the write path
**What you see:** A traffic spike on product pages makes checkout slow, because reads and writes compete for the same CPU, memory, and buffer pool.

**Why it happens:** One node serves everything, and most workloads are read-dominated — frequently 90-99% reads.

**How replication solves it:** Point reads at replicas and writes at the primary. Add replicas to add read capacity, near-linearly. This is the cheapest way to scale a read-heavy system and should be tried long before sharding, which is far more invasive.

### 3. Downtime during maintenance
**What you see:** Every database upgrade, schema change, or OS patch means a maintenance window.

**Why it happens:** Work on the only node means downtime on the only node.

**How replication solves it:** Upgrade a replica, verify it, then fail over to it. Maintenance becomes a controlled switch rather than an outage. The same mechanism enables running expensive analytics and backups on a replica so they never touch the machine serving customers.

### 4. Latency for users on the other side of the world
**What you see:** Users in Singapore wait 250 ms per query because the database is in Virginia.

**Why it happens:** Physics. Round-trip time across an ocean has a floor.

**How replication solves it:** Geographically distributed read replicas serve local reads locally. Writes still travel to the primary, which is why write-heavy global apps eventually need multi-primary or multi-region designs.

## The Price You Pay

This is where replication gets genuinely interesting, and where interviews probe:

- **Replication lag and stale reads.** Asynchronous replication means a replica can be milliseconds — or, under load, many seconds — behind. A user updates their profile, the write goes to the primary, the redirect reads from a replica, and their change appears to have vanished. This "read-your-own-writes" violation is the classic bug, and the fixes (route a user's reads to the primary briefly after a write, or use read-your-writes tokens) all add complexity.
- **Synchronous replication costs latency.** You can eliminate lag by waiting for replicas to acknowledge, but now every write pays the network round trip to the slowest replica, and a slow replica degrades the whole system. Semi-synchronous (wait for one of several) is the common compromise.
- **Failover is hard and dangerous.** Detecting that a primary is truly dead versus briefly unreachable is genuinely difficult. Get it wrong and you get **split-brain**: two nodes both accepting writes, diverging, with no clean way to merge afterward. This is why serious setups use consensus-based leader election rather than a timeout and a hope.
- **Multi-primary means conflicts.** Accepting writes on two nodes means two users can update the same row simultaneously in different places. Somebody has to resolve that — last-write-wins (silently loses data), application-level merge (complex), or CRDTs (specialized). Master-master is far less of a free lunch than it looks.
- **Cost.** Each replica is a full copy of your data on real hardware.

## When You Need It — and When You Don't

| Replicate when | Hold off when |
|---|---|
| Data loss is unacceptable (essentially always, in production) | It's a dev/staging environment |
| The workload is read-heavy and reads are the bottleneck | Writes are the bottleneck — you need sharding, not replicas |
| You need failover without a long restore | Data is fully reconstructible from another source |
| Users are geographically spread | Everyone is in one region and latency is fine |

## Why This Shows Up in Interviews

Replication is where interviewers test whether you understand *consequences*, not just components. Anyone can say "add read replicas." The follow-ups separate people: what happens when a user reads their own write from a lagging replica? How do you detect primary failure? What is split-brain and how do you avoid it? What does synchronous replication cost you? Volunteering the read-your-own-writes problem before being asked is one of the clearest seniority signals available in a database discussion.

## How It Connects

Replication is the concrete mechanism behind **fault tolerance** and **availability** at the data layer. It's the direct cause of **eventual consistency** in most real systems, which makes it the practical face of **CAP theorem**. Safe failover requires **consensus algorithms** (Raft/Paxos). It pairs with — and is distinct from — **sharding**: replication copies the same data for availability and read scale; sharding splits different data for write scale. Real systems use both, and **multi-region architecture** is replication at the largest scale.

**Next:** [Database Sharding & Partitioning](../14-database-sharding-and-partitioning/why.md) — for when the bottleneck is writes, not reads.
