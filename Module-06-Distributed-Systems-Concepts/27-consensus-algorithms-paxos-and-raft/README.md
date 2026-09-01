# Consensus Algorithms: Paxos & Raft

**Difficulty:** Advanced | **Estimated length:** 20-25 min | **Prerequisites:** [Circuit Breaker, Retry & Bulkhead Patterns](../26-circuit-breaker-retry-and-bulkhead-patterns/README.md), [Database Replication](../../Module-03-Databases-and-Storage/13-database-replication/README.md), [CAP Theorem and PACELC](../../Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md)

## Learning Objectives

- Explain why distributed consensus is fundamentally hard, including the intuition behind the FLP impossibility result.
- Describe Paxos's roles (Proposer, Acceptor, Learner) and its two phases (Prepare/Promise, Accept/Accepted), and why a majority quorum guarantees safety.
- Describe Raft's roles (Leader, Follower, Candidate), terms, leader election, and log replication, including the commit rule.
- Compare Paxos and Raft on understandability, structure, and real-world adoption.
- Identify which production systems rely on each algorithm (or a variant) and why.

## Script

### Hook / Intro

Imagine five people running a company remotely, none of them fully trusting the phone lines between them. Calls drop. Someone might freeze mid-sentence and never speak again. Two people might both think they're in charge at the same time. And yet, this group has to agree — reliably, unanimously, and irreversibly — on a single decision, like "who is CEO" or "what is the next line item in the ledger." That is distributed consensus, and it's one of the deepest problems in computer science. Today we're tackling the two algorithms that solve it in practice: Paxos, the original and notoriously difficult solution, and Raft, the algorithm designed specifically to be understandable. Fair warning — this is one of the hardest topics in this entire course. We're not going to oversimplify it. Let's get into it.

### Why Consensus is Hard

Consensus means getting a set of distributed nodes to agree on a single value, even when some nodes fail or messages get delayed, lost, or reordered. Sounds simple. It isn't. In 1985, Fischer, Lynch, and Paterson proved a result now called the FLP impossibility: in a fully asynchronous network — one with no bound on message delay — no deterministic consensus algorithm can guarantee both safety and termination if even one node can fail. In plain terms: you cannot build a protocol that is always correct AND always finishes, when you can't tell the difference between "a node is slow" and "a node is dead."

Real systems get around this by relaxing assumptions — Paxos and Raft assume the network is "partially synchronous" in practice, meaning it's mostly reliable most of the time, and they use timeouts to make progress even though, in theory, an adversarial network could stall them forever. That's an acceptable trade-off because real networks aren't adversarial forever.

Then there's the failure model itself. Nodes crash. Networks partition — a switch fails, and your five-node cluster splits into a group of three and a group of two, each side still able to talk internally but not across the split. If both sides kept operating independently and accepting writes, you'd get split-brain: two subsets of the same system, each believing it's authoritative, diverging into inconsistent states. Consensus algorithms exist specifically to prevent split-brain by requiring decisions to be backed by a majority quorum — more than half the nodes — so that two disjoint majorities can never exist at the same time in a cluster of a fixed size.

### Paxos — The Original Solution

Paxos was described by Leslie Lamport in the late 1980s, published in 1998, and it's the foundational proof that consensus is achievable despite FLP, given majority quorums and enough eventual synchrony.

Paxos defines three roles. The Proposer suggests a value to be agreed upon. The Acceptor votes on proposals and forms the persistent memory of the system. The Learner finds out what value was chosen. In practice, one physical node often plays multiple roles.

Paxos runs in two phases, and getting the phase numbering right matters because it shows up in every real implementation. Phase 1a is Prepare: a Proposer picks a proposal number — unique and monotonically increasing — and sends a Prepare request to a majority of Acceptors. Phase 1b is Promise: each Acceptor that receives a Prepare with a number higher than anything it has seen before replies with a Promise, meaning "I won't accept any proposal numbered lower than this," and it includes the highest-numbered proposal it has already accepted, if any. Phase 2a is Accept: once the Proposer hears Promises from a majority, it sends an Accept request with a value — either its own value, or, critically, the value from the highest-numbered proposal any Acceptor already reported, so the algorithm never overwrites a value that might already be chosen. Phase 2b is Accepted: each Acceptor that hasn't promised something higher in the meantime accepts, and once a majority accepts the same value, that value is chosen — permanently.

Why does a majority quorum guarantee safety? Because any two majorities out of the same set of nodes must overlap by at least one node. That overlapping Acceptor is the one that carries forward the previously accepted value in Phase 1b, which is what prevents two different values from ever being chosen for the same instance.

Here's the honest part: Paxos, as specified, is famously hard to understand and even harder to implement correctly. Lamport's own paper is dense enough that he later wrote "Paxos Made Simple" just to re-explain it. Single-instance Paxos also only agrees on one value; real systems need a continuous log of decisions, which is why practical deployments use Multi-Paxos — a leader-like optimization that skips Phase 1 for a stream of instances once one Proposer is established as stable, dramatically cutting message overhead.

### Raft — Designed for Understandability

Raft was published in 2014 by Diego Ongaro and John Ousterhout with an explicit design goal: solve the exact same problem as Multi-Paxos, but be understandable enough to teach and implement correctly. They succeeded — Raft is now the default choice for new systems.

Raft decomposes consensus into three clearer sub-problems: leader election, log replication, and safety. Nodes have one of three roles at any time: Follower, Candidate, or Leader. Time is divided into terms — logical, monotonically increasing epochs, each with at most one Leader.

Leader election works like this: every Follower has a randomized election timeout. If it hears no heartbeat from a Leader before that timeout expires, it becomes a Candidate, increments the term number, votes for itself, and requests votes from every other node. If it receives votes from a majority of the cluster, it becomes Leader for that term and starts sending periodic heartbeats to suppress further elections. The randomization of timeouts is the key trick — it makes it statistically unlikely that two nodes become candidates simultaneously and split the vote, so elections converge quickly in practice.

Log replication is how the Leader gets work done. Clients send commands to the Leader, which appends each command as a new entry to its local log and then replicates that entry to Followers in parallel. Once the Leader confirms that a majority of nodes — including itself — have durably appended the entry, it commits the entry, applies it to its state machine, and responds to the client. It then notifies Followers of the new commit index on the next heartbeat so they can apply it too. Followers that are behind get brought up to date by the Leader forcing its log onto them, guaranteeing log consistency across the cluster.

On safety: Raft guarantees that a candidate can only win an election if its log is at least as up to date as a majority of the cluster's logs — comparing the term and index of the last log entry — which ensures a newly elected Leader never overwrites already-committed entries. This is a much more explicit, structurally enforced version of the same guarantee Paxos achieves implicitly through proposal numbers.

### Paxos vs Raft

Both algorithms solve the same problem and both rely on majority quorums for safety, so mathematically they're equivalent in what they can guarantee. The differences are in structure and ergonomics. Paxos is symmetric and leaderless by default, which is elegant in theory but leaves huge gaps — like leader election and log management — as "engineering exercises" for anyone implementing it. Raft bakes leader election and a strongly leader-based log replication model directly into the specification, which makes the whole system easier to reason about, easier to test, and easier to get right the first time. That's precisely why Raft has become the go-to choice for new distributed systems, even though Paxos got there first and proved it was possible.

### Real-World Example

Paxos and its variants power some very large systems. Google's Chubby lock service runs a Paxos-based replicated state machine, and Google Spanner uses Paxos to replicate its transaction logs across data centers. Apache ZooKeeper uses ZAB — the ZooKeeper Atomic Broadcast protocol — which is Paxos-like in spirit, tailored for primary-backup replication of a sequence of state updates. On the Raft side, etcd — the coordination store behind Kubernetes — implements Raft directly and it's the reason Kubernetes can survive control-plane node failures without losing cluster state. HashiCorp Consul and CockroachDB, a distributed SQL database, also use Raft for their replication layers, precisely because Raft's clarity made it feasible for their teams to implement and operate confidently.

### Recap

Consensus lets a distributed system agree on a value despite failures and network unreliability, and FLP tells us this is theoretically impossible to do perfectly — so both Paxos and Raft use majority quorums and timeouts to make it work in practice. Paxos, with Proposers, Acceptors, and Learners running Prepare/Promise and Accept/Accepted phases, was the first proof that this is achievable, but it's hard to implement. Raft restructures the same guarantees around an explicit Leader, terms, randomized election timeouts, and majority-acknowledged log replication, trading a little theoretical elegance for a lot of practical clarity — which is why it now backs etcd, Consul, and CockroachDB.

### What's Next

Consensus gets a single log of operations agreed upon across replicas — but what happens when a single business transaction needs to update data across multiple independent services or databases? That's a different coordination problem. Next time, we're covering Distributed Transactions: Two-Phase Commit and the Saga pattern.

## Key Takeaways

- Consensus is provably hard (FLP impossibility) in a fully asynchronous network with even one faulty node; real systems rely on partial synchrony and timeouts to make progress anyway.
- Majority quorums are the core safety mechanism in both Paxos and Raft: any two majorities in the same cluster must overlap, which prevents split-brain and conflicting decisions.
- Paxos uses Proposers, Acceptors, and Learners across Phase 1 (Prepare/Promise) and Phase 2 (Accept/Accepted); it's provably correct but notoriously hard to implement, so real deployments use Multi-Paxos.
- Raft splits consensus into leader election, log replication, and safety, using terms and randomized election timeouts to elect a single Leader that drives all log writes.
- A Raft log entry commits once a majority of nodes (including the Leader) have durably appended it — in an N-node cluster, you need floor(N/2) + 1 nodes.
- Paxos and Raft are equally powerful in theory; Raft won broader real-world adoption (etcd, Consul, CockroachDB) because its explicit leader-based design is far easier to implement and operate correctly.
