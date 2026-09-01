# Practice & Interview Questions

**1. What problem does a consensus algorithm actually solve, and why can't you just have every node "agree by majority vote" without a formal protocol?**
Consensus gets a set of distributed nodes to agree on a single value (or an ordered sequence of values) despite node failures and unreliable message delivery. A naive majority vote breaks down because messages can be delayed, reordered, or lost, and nodes can fail mid-decision — without a formal protocol (proposal numbers/terms, quorum overlap rules, and defined phases), you can end up with two different values both appearing "chosen" to different parts of the cluster, which is exactly the safety violation consensus algorithms are built to prevent.

**2. What does the FLP impossibility result actually say, and how do Paxos and Raft get around it in practice?**
FLP (Fischer, Lynch, Paterson, 1985) proves that in a fully asynchronous network with no bound on message delay, no deterministic algorithm can guarantee both safety and termination if even one node can fail — you cannot always tell a slow node from a dead one. Paxos and Raft don't disprove this; they sidestep it by assuming partial synchrony in practice (messages usually arrive within some reasonable time) and using timeouts to trigger retries or new elections, accepting that in a pathological network they could theoretically stall, though this essentially never happens in real deployments.

**3. In a 7-node Raft cluster, how many nodes must acknowledge a log entry before it's committed, and why?**
4 nodes (including the Leader itself) — a majority, computed as `floor(7/2) + 1 = 4`. This is required because any two majority sets among 7 nodes must overlap in at least one node; that guaranteed overlap is what ensures a future Leader (elected by a different majority) will always see the previously committed entry and cannot silently discard it.

**4. What happens during a network partition that splits a 5-node Raft cluster into a 3-node group and a 2-node group?**
The 3-node side still has a majority (3 out of 5), so it can elect/keep a Leader and continue committing new log entries normally. The 2-node side cannot form a majority, so any node there that becomes a Candidate will keep timing out without winning an election, and no writes can be committed on that side. This is by design: it sacrifices availability on the minority side to preserve consistency and prevent split-brain.

**5. Why is Paxos considered notoriously hard to implement correctly, despite being provably correct?**
The base Paxos protocol only specifies how to agree on a single value and deliberately leaves out practical concerns — how to efficiently run a continuous sequence of decisions, how to pick a stable proposer to avoid dueling proposers, how to handle log compaction and membership changes. Implementers have to invent solutions to all of this themselves (often as "Multi-Paxos" variants), and subtle mistakes in those extensions are easy to make and hard to catch, which is exactly the gap Raft was designed to close by specifying these mechanisms explicitly.

**6. Walk through what happens, step by step, when a Raft Follower's election timeout expires.**
- It transitions to Candidate state and increments the current term number.
- It votes for itself and resets its own election timer with a new random timeout.
- It sends RequestVote RPCs to all other nodes, including its last log index/term.
- If it receives votes from a majority of the cluster, it becomes Leader for that term and starts sending heartbeats.
- If it discovers another node is already Leader for an equal or higher term, or if the vote splits and no one wins, it reverts to (or stays) Follower/Candidate and the process retries with a new randomized timeout.

**7. Why does Raft use randomized election timeouts instead of a fixed timeout for every node?**
If every Follower used the same fixed timeout, many nodes would likely become Candidates at the exact same moment after a Leader failure, splitting the vote repeatedly and delaying election convergence indefinitely. Randomizing the timeout across a range makes it statistically likely that one node times out first, becomes Candidate, and gathers a majority before others even start their own election, so elections resolve quickly in almost all cases.

**8. In Paxos, why must a Proposer, upon receiving Promise responses, propose the value from the highest-numbered proposal already accepted by any Acceptor, rather than its own preferred value?**
If an Acceptor's Promise response indicates it already accepted some value V under an earlier proposal number, that value V might already be on its way to being chosen by a majority (or might already have been chosen, with the Proposer simply unaware). Proposing anything other than V risks two different values being chosen for the same instance, which would violate consensus safety — so the Proposer is forced to "adopt" the highest-numbered previously accepted value to preserve the invariant that at most one value can ever be chosen.

**9. What are the three sub-problems Raft explicitly decomposes consensus into, and why does that decomposition make Raft easier to reason about than Paxos?**
Leader election, log replication, and safety. By assigning each concern its own clearly specified mechanism (randomized-timeout elections; Leader-driven AppendEntries replication; an explicit election restriction plus commit rule for safety), Raft avoids the ambiguity of "generic" Paxos, where all of these concerns are entangled in the abstract Proposer/Acceptor model and left for implementers to work out on their own.

**10. Name one real-world system that uses Paxos (or a Paxos-like protocol) and one that uses Raft. Why might a team choose one over the other today?**
Google Spanner and Chubby use Paxos-based replication, and ZooKeeper uses ZAB, a Paxos-like protocol; etcd, Consul, and CockroachDB use Raft. A team building a new system today would generally lean toward Raft because its explicit leader election and log replication rules are far easier to implement correctly, test, and debug — Paxos is chosen mainly by teams inheriting an existing Paxos-based system or optimizing an already mature implementation.

**11. What is split-brain, and specifically how does the majority-quorum requirement prevent it?**
Split-brain is when a network partition causes two or more subsets of a cluster to each act as if they are the sole authority, independently accepting writes and causing the system's state to diverge. Because a majority quorum requires more than half of a fixed-size cluster, and any two majority subsets of the same cluster must share at least one node, it is mathematically impossible for two disjoint groups to simultaneously both claim a majority — so at most one side of any partition can make progress at a time.

**12. In Raft, could a node with a stale (out-of-date) log ever win a leader election? Explain the safety mechanism that prevents this.**
No — Raft's election restriction requires that a voter reject a RequestVote request if the candidate's log is less up to date than its own (compared first by the term of the last log entry, then by index). A candidate can only win by getting votes from a majority, and since a majority of nodes must hold every previously committed entry, at least one voter in that majority would already have the most up-to-date log — meaning it would refuse to vote for a candidate lagging behind, which guarantees a stale node cannot accumulate the votes needed to become Leader.
