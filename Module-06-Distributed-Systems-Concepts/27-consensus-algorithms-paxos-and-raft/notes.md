# Study Notes: Consensus Algorithms (Paxos & Raft)

## Core Definitions

- **Consensus**: The problem of getting a set of distributed nodes to agree on a single value or sequence of values, despite node failures, message loss, delay, or reordering.
- **Quorum**: The minimum number of nodes that must participate in a decision for it to be considered valid. A **majority quorum** in an N-node cluster is `floor(N/2) + 1` nodes. Any two majority quorums drawn from the same cluster must share at least one common node — this overlap is what guarantees safety.
- **Split-brain**: A failure mode where a network partition causes two (or more) disjoint subsets of a cluster to each believe they are authoritative and independently accept writes, leading to divergent, inconsistent state. Majority-quorum consensus prevents this because only one side of a partition (at most) can ever contain a majority.
- **Leader election**: The process by which a distributed cluster selects a single coordinating node (the Leader) responsible for driving decisions/writes for a period of time, typically triggered when no current Leader is known or the previous Leader is presumed failed.
- **Log replication**: The mechanism by which a Leader (or Proposer) propagates an ordered sequence of operations/entries to other nodes so that all replicas converge on the same sequence of committed state changes.
- **Term / Epoch**: A monotonically increasing logical clock value used to detect stale leaders/proposals. Raft calls this a "term" (at most one Leader per term). Paxos achieves the same effect with monotonically increasing, globally unique **proposal numbers**.
- **FLP impossibility (1985, Fischer–Lynch–Paterson)**: In a fully asynchronous network model with no bound on message delay, no deterministic consensus algorithm can guarantee both safety and termination (liveness) if even a single node may fail. Real systems sidestep this with partial synchrony assumptions and timeouts, trading theoretical guarantees for practical progress.

## Paxos vs Raft — Comparison Table

| Aspect | Paxos | Raft |
|---|---|---|
| Roles | Proposer, Acceptor, Learner | Leader, Follower, Candidate |
| Structure | Symmetric, leaderless by default | Explicitly leader-based (strong leader) |
| Phases | Phase 1 Prepare/Promise, Phase 2 Accept/Accepted | Leader election (RequestVote), Log replication (AppendEntries) |
| Ordering primitive | Proposal number (unique, monotonic) | Term number + log index |
| Understandability | Notoriously difficult; original paper needed a follow-up ("Paxos Made Simple") | Explicitly designed for understandability (Ongaro & Ousterhout, 2014) |
| Practical variant | Multi-Paxos (skips Phase 1 across a stream of instances once a stable proposer/leader exists) | Raft itself is already a multi-entry log protocol |
| Leader-based? | Not inherently; Multi-Paxos adds a de facto leader for efficiency | Yes, by design — all writes go through the current Leader |
| Safety mechanism | Majority quorum overlap + carrying forward highest-numbered accepted value | Majority quorum overlap + "leader completeness" (candidate's log must be at least as up to date as a majority) |
| Real-world adoption | Google Chubby, Google Spanner (Paxos-based log replication); ZooKeeper's ZAB is Paxos-like | etcd (Kubernetes), HashiCorp Consul, CockroachDB |

## Raft's Three Sub-Problems

- **Leader election** — exactly one Leader per term, elected via randomized election timeouts and majority vote (RequestVote RPC).
- **Log replication** — the Leader appends client commands to its log and replicates them to Followers (AppendEntries RPC); an entry commits once a majority of nodes have durably stored it.
- **Safety** — guarantees that once an entry is committed, no future Leader can overwrite or lose it; enforced via the election restriction (a candidate must have an up-to-date log to win) and the commit rule (only commit entries from the current term directly; earlier terms commit transitively).

## Key Numbers

- Majority = `floor(N/2) + 1`.
- 3-node cluster: majority = 2, tolerates 1 node failure.
- 5-node cluster: majority = 3, tolerates 2 node failures.
- 7-node cluster: majority = 4, tolerates 3 node failures.
- Adding nodes beyond an odd number without increasing fault tolerance is wasteful — e.g., a 4-node cluster still only tolerates 1 failure (majority = 3) but pays more replication cost than a 3-node cluster with the same fault tolerance. This is why production Raft/Paxos clusters are almost always sized as odd numbers (3, 5, 7).

## Interview Revision — Quick Bullets

- Consensus = agreeing on one value/log across unreliable nodes; both Paxos and Raft rely on majority quorums for safety, not unanimity.
- FLP impossibility says perfect consensus (safety + guaranteed termination) is impossible in a purely asynchronous model with failures — real systems use timeouts and partial synchrony to work around this.
- Paxos phases: Phase 1a Prepare / Phase 1b Promise, then Phase 2a Accept / Phase 2b Accepted. A value is chosen once a majority of Acceptors accept it.
- Raft phases: Leader election (terms, randomized timeouts, votes) then log replication (AppendEntries, commit on majority ack).
- Split-brain is prevented because two disjoint majorities cannot exist simultaneously in one fixed-size cluster.
- Raft won adoption over raw Paxos because it is far easier to implement correctly, not because it is theoretically stronger — both are equally powerful.
- Know one production example for each: etcd/Consul/CockroachDB → Raft; Chubby/Spanner → Paxos-variant; ZooKeeper → ZAB (Paxos-like).
