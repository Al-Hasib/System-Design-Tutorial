# Why This Topic Matters: Consensus Algorithms (Paxos & Raft)

> **In one sentence:** Every distributed system eventually needs a group of machines to agree on a single value — who the leader is, what the config says, whether a lock is held — and "just take a vote" doesn't work when messages can be lost, delayed, or arrive out of order.

## The World Before This Idea

You have three database replicas and the primary stops responding. The other two need to pick a new primary. How?

Try the naive approaches and watch each one fail:

- **Timeout and self-promote.** Both replicas time out, both promote themselves. Now two primaries accept writes, they diverge, and when the network heals there is no correct way to merge them. This is **split-brain**, and it means permanent data loss.
- **Ask a coordinator.** The coordinator is now a single point of failure. You've moved the problem, not solved it.
- **Majority vote.** Closer — but what if messages are delayed rather than lost? A node can vote, crash, restart having forgotten, and vote again. Two nodes can each believe they won.

The reason this is hard is that in an asynchronous network, **you cannot distinguish a crashed node from a slow one**. A node that hasn't replied might be dead, or might be about to reply. Any protocol that assumes otherwise is broken, and it will be broken in production at the worst possible time.

## The Problems It Solves

### 1. Split-brain and the permanent divergence it causes
**What you see:** After a network partition heals, two nodes have accepted conflicting writes. Data is irreconcilably wrong.

**Why it happens:** Both sides of a partition believed they were in charge.

**How consensus solves it:** A leader is elected only with a **majority quorum** (more than half the nodes). Because two majorities of the same set must overlap in at least one node, and that node won't vote twice in the same term, two leaders cannot be elected simultaneously. The minority side *cannot* elect a leader and therefore cannot accept writes. Split-brain is prevented by arithmetic, not by hope.

### 2. Configuration that different nodes disagree about
**What you see:** Half your fleet has the old shard map, half has the new one, and requests route to the wrong places.

**Why it happens:** Distributing config by pushing it to nodes individually has no atomicity — some nodes get it, some don't, some get it late.

**How consensus solves it:** A replicated state machine. All nodes apply the same commands in the same order from a consensus-maintained log, so every node's state is identical. This is exactly what ZooKeeper and etcd are: small, highly reliable, consensus-backed stores for the facts everyone must agree on. Kubernetes stores its entire cluster state in etcd for this reason.

### 3. Locks and leases that aren't actually safe
**What you see:** Two workers both believe they hold the lock and both process the same job, double-charging a customer.

**Why it happens:** A lock in a non-consensus store (a single Redis node, say) can be lost on failover, or granted twice during a partition.

**How consensus solves it:** A consensus-backed lock service gives a genuine guarantee, plus fencing tokens — monotonically increasing numbers that let downstream systems reject a request from a stale lock holder.

### 4. "Just use a leader" without a way to elect one
**What you see:** Architectures that assume a single writer, coordinator, or scheduler, with no defined answer for what happens when it dies.

**Why it happens:** Leader-based designs are much simpler to reason about, so they're chosen — and the election problem is deferred.

**How consensus solves it:** Raft in particular makes leader election a first-class, understandable mechanism (terms, election timeouts, heartbeats). This is why Raft displaced Paxos in practice: Paxos is correct but famously hard to understand and even harder to implement correctly, while Raft was explicitly designed for comprehensibility. That design goal is itself an engineering lesson.

## The Price You Pay

Consensus is expensive, which is exactly why you use it sparingly:

- **Every write costs a round trip to a majority.** Latency is bounded below by your slowest majority member, which across regions means tens or hundreds of milliseconds per operation. This is not a high-throughput data path.
- **Throughput doesn't scale with nodes — it gets worse.** More nodes means more messages per decision. Five is the typical sweet spot; nine is usually worse than five.
- **It sacrifices availability by design.** With a majority requirement, losing a majority means the system stops accepting writes entirely. That's the correct behavior (it's the CP choice in CAP), and it means a consensus system will deliberately go down rather than risk inconsistency.
- **Implementing it yourself is a bad idea.** Correct consensus implementations take years to harden. Use etcd, ZooKeeper, Consul, or a database that embeds a well-tested implementation.

**The practical rule:** use consensus for small amounts of critical metadata — leadership, membership, configuration, locks — and keep your bulk data path out of it.

## When You Need It — and When You Don't

| You need consensus when | You don't when |
|---|---|
| Exactly one node must be leader | Nodes are stateless and interchangeable |
| Correctness beats availability (financial, inventory) | Eventual consistency is acceptable |
| Cluster membership or config must be globally agreed | Conflicts can be resolved after the fact (CRDTs, LWW) |
| You need genuinely safe distributed locks | A coordination service already handles it for you |

## Why This Shows Up in Interviews

Consensus is a senior-level topic and interviewers use it to probe depth. You're unlikely to be asked to implement Raft, but you should be able to explain: why a naive timeout-and-promote scheme causes split-brain, why a majority quorum prevents two leaders, why consensus is too slow for the main data path, and which real systems use it (etcd, ZooKeeper, Consul, Kafka's controller, CockroachDB, Spanner). Knowing that consensus is deliberately CP — that it chooses to stop rather than diverge — connects it cleanly to CAP and reads as genuine understanding.

## How It Connects

Consensus is the CP corner of **CAP** (topic 15) made concrete. It's what makes **replication** failover safe rather than dangerous (topic 13), the foundation of correct **distributed locking** (topic 40), and the mechanism behind cluster membership that **consistent hashing** (topic 24) and **service discovery** (topic 31) depend on. It's also the machinery underneath the coordinator in **two-phase commit** (topic 28) and the control plane of **Kubernetes** (topic 44).

**Next:** [Distributed Transactions: 2PC & Saga](../28-distributed-transactions-2pc-and-saga/why.md) — agreeing not on a value, but on whether a multi-service operation happened.
