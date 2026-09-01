# CAP Theorem & PACELC Explained

**Difficulty:** Intermediate/Advanced
**Estimated length:** 12-18 min
**Prerequisites:** [Database Replication: Master-Slave & Master-Master](../13-database-replication/README.md), [Database Sharding & Partitioning Strategies](../14-database-sharding-and-partitioning/README.md)

## Learning Objectives

By the end of this video, you should be able to:

- Define Consistency, Availability, and Partition Tolerance the way the CAP theorem actually uses those words.
- Explain why partition tolerance isn't really a choice in a real distributed system, and why CAP effectively boils down to CP vs. AP.
- Classify real databases (MongoDB, HBase, ZooKeeper, Cassandra, DynamoDB) as leaning CP or AP, with reasons.
- Explain the PACELC extension and why it's a more complete model than CAP alone.
- Apply these trade-offs to a real scenario, like a shopping cart or a banking ledger.

## Script

### Hook/Intro

Okay. So in the last two videos, we did something kind of dangerous. We took our data and we spread it out. In the replication video, we made copies of it across multiple machines. In the sharding video, we split it into pieces across multiple machines. Both of those techniques make a system more scalable and more resilient — but they both share the same hidden cost: now your data lives in more than one place, connected by a network. And networks fail.

That single fact — that networks are unreliable — is the doorway into one of the most misunderstood, most quoted, and most interview-relevant ideas in all of distributed systems: the CAP theorem. And, as we'll see, CAP is actually the *simplified* version of the story. There's a more nuanced, more practically useful extension called PACELC that most engineers never learn but every engineer should. Let's get into it.

### The CAP Theorem

CAP theorem was put forward by Eric Brewer around 2000, and later formally proven. It says that any distributed data store can only provide two out of these three guarantees at the same time:

**Consistency** — every read receives the most recent write, or an error. Not "eventually correct" — every single node you ask gives you the same, up-to-date answer, as if there were only one copy of the data.

**Availability** — every request that hits a working node receives a response. Not an error, not a timeout — a real response, even if the system is in a degraded state.

**Partition Tolerance** — the system continues to operate even when the network between nodes drops messages or gets cut entirely. Think of a partition as a chasm opening up between two data centers — they simply can't talk to each other for a while.

Here's the classic "pick two" pitch: C+A, C+P, or A+P. But here's the thing that trips people up, and it's the single most important insight in this entire video: partition tolerance isn't really optional. In any distributed system — meaning more than one node, talking over an actual network — partitions *will* happen eventually. A cable gets cut, a router misbehaves, a data center loses connectivity, a process just gets slow enough that it looks unreachable. If you're building a distributed system at all, you have to tolerate that reality; you don't get to opt out of P by wishing really hard.

So the real, practical choice isn't "which two of three" — it's this: **when a partition actually happens, do you sacrifice Consistency or do you sacrifice Availability?** That's why people really say CAP is CP vs. AP. C+A without P only exists in the fantasy world of a single machine, or a network that never fails — and we don't build systems on fantasies.

### CP Systems in Depth

A CP system, when a partition hits, chooses to preserve consistency and gives up availability. Concretely: if a node can't confirm with the rest of the cluster that it has the latest data, it will refuse to answer rather than risk giving you something stale or wrong. It would rather say "sorry, I can't help you right now" than lie to you.

Good examples: ZooKeeper, etcd, and HBase are classic CP systems — they're often used specifically *because* they're strongly consistent, for things like leader election and configuration management, where two nodes disagreeing about the truth would be catastrophic. MongoDB, in its default configuration with a majority write concern and read concern, also leans CP — if a primary can't reach a majority of replicas, it steps down rather than continuing to serve writes that might get lost.

The mental model: think of a bank vault with a single combination that requires the manager and the assistant manager both present to open it. If the assistant manager is unreachable, the vault stays locked, even if that's inconvenient. Correctness over convenience.

### AP Systems in Depth

An AP system does the opposite: when a partition hits, it chooses to keep answering requests, even if that means different nodes might give you different, possibly stale, answers. It would rather say "here's an answer" — even a slightly outdated one — than refuse to respond.

Cassandra and DynamoDB are the textbook AP examples. Both were explicitly designed for "always-on" availability — Amazon literally built Dynamo (the paper behind DynamoDB) because a partition-related outage once meant customers couldn't add items to their shopping cart, and losing sales was worse than occasionally reconciling conflicting cart data later. Both systems let you keep reading and writing during a partition, and they resolve conflicts afterward using things like vector clocks, last-write-wins, or version reconciliation.

Mental model: think of a chain of convenience stores that keeps selling gift cards during an internet outage using a local paper ledger, and reconciles balances with headquarters once the connection is back. Customers get served. There is some risk of a double-spend somewhere, but the business decided that risk was worth it.

### Common Misconceptions

Here's the one that catches almost everyone: CAP theorem is **only** about behavior during an actual network partition. It says nothing about how a system behaves the rest of the time — which, for most well-run systems, is the vast majority of the time. A system isn't "a CP system" or "an AP system" in some permanent, all-day sense; it's making a specific choice about what to sacrifice *at the moment a partition occurs*. During normal operation, with no partition, you can absolutely have both consistency and availability. CAP just tells you what breaks when the network breaks.

That gap — "what happens the other 99.9% of the time when there's no partition?" — is exactly what CAP theorem doesn't answer. Which is exactly the gap PACELC was designed to fill.

### Introducing PACELC

PACELC, coined by Daniel Abadi, extends the idea like this:

**P**artition — if there's a partition, choose between **A**vailability and **C**onsistency (this part is just CAP).

**E**lse — if there's no partition, meaning the system is running normally, choose between **L**atency and **C**onsistency.

That second half is the missing piece. Even with a perfectly healthy network, there's still a trade-off: do you wait for all replicas to acknowledge a write before confirming it (higher consistency, higher latency), or do you confirm as soon as one node has it and sync the rest in the background (lower latency, weaker consistency guarantees)? That's a real, everyday engineering decision, completely separate from partitions.

This is why PACELC is considered more complete and more useful in practice: it captures both the rare, dramatic failure-mode trade-off (P) and the constant, everyday trade-off (E) that's happening on every single request your system serves.

Let's place some real systems on the PACELC map:

- **DynamoDB** is PA/EL — during a partition it favors availability, and during normal operation it favors low latency over strict consistency (though it does offer an optional strongly-consistent read mode if you're willing to pay the latency cost).
- **Cassandra** is tunable, but by default leans PA/EL too — you can dial the consistency level per query (ONE, QUORUM, ALL) and directly trade latency for consistency.
- **MongoDB**, configured for majority write/read concern, leans PC/EC — it favors consistency both during a partition and during normal operation, accepting extra latency and reduced availability as the cost.

### Real-World Example

Picture a global e-commerce "add to cart" service, replicated across regions. A network partition splits the US and EU data centers for two minutes. If this system is AP, a shopper in Paris can keep adding items — their request hits the EU replica, which just answers locally. Maybe, in a rare edge case, an item that just sold out on the US side gets added to a cart anyway, and that gets reconciled at checkout. Annoying, but survivable, and the business kept selling.

Now picture a bank transferring money between two accounts. If that system stays available during a partition, you risk letting the same balance be spent twice from two disconnected replicas — that's not an annoying edge case, that's a real financial and regulatory problem. Here, a bank will typically choose CP: during the partition, the affected transaction simply fails or blocks rather than risk an inconsistent balance. Same underlying theorem, completely different choice, because the cost of being wrong is completely different.

### Recap

Let's tie it together. CAP theorem says that during a network partition, a distributed system must choose between Consistency and Availability — partition tolerance itself isn't optional, so it's really a CP vs. AP decision. CP systems refuse to answer rather than risk staleness; AP systems keep answering and reconcile later. But CAP only describes the rare partition scenario. PACELC fills in the rest: even without a partition, you're always trading Latency against Consistency. Real systems like DynamoDB, Cassandra, and MongoDB sit at different points on that PA/EL to PC/EC spectrum, and which point is "right" depends entirely on what it costs your business to be wrong versus what it costs to be slow or unavailable.

### What's Next

Now that we understand *why* distributed systems can't give us perfect consistency and perfect availability at the same time, the natural next question is: how do database transaction models actually formalize these trade-offs? In the next video, we'll dig into ACID vs. BASE — the transaction guarantee models that grew directly out of this consistency-versus-availability tension — along with normalization and denormalization, the data-modeling decisions that go hand in hand with them. See you there.

## Key Takeaways

- CAP theorem: pick two of Consistency, Availability, Partition tolerance — but since partitions are unavoidable in real distributed systems, the practical choice is CP vs. AP.
- CP systems (ZooKeeper, etcd, HBase, MongoDB with majority concerns) sacrifice availability during a partition to guarantee correctness.
- AP systems (Cassandra, DynamoDB) sacrifice strict consistency during a partition to keep serving requests, reconciling conflicts later.
- CAP only describes behavior *during* an active network partition — it says nothing about the much more common case of normal operation.
- PACELC extends CAP: during a Partition choose Availability or Consistency, Else (normal operation) choose Latency or Consistency — giving a fuller picture of a system's design trade-offs.
- The "right" choice is a business decision, not a technical one: weigh the cost of stale/incorrect data against the cost of downtime or slowness.
