# Practice & Interview Questions

**1. What do the three letters in CAP theorem stand for, and what does each actually guarantee?**
- Consistency: every read returns the most recent write (or an error), as if there were a single copy of the data.
- Availability: every request to a non-failed node gets a response, though not necessarily the latest data.
- Partition Tolerance: the system keeps operating even when network messages between nodes are dropped or delayed.

**2. Does the CAP theorem mean a system is "CP" or "AP" at all times?**
No. CAP theorem only describes what a system does *during an active network partition*. During normal operation, with a healthy network, a well-designed system can provide both consistency and availability. Calling a system "CP" or "AP" is shorthand for what it chooses to sacrifice specifically when a partition occurs.

**3. Why is "CA" (Consistency + Availability, without Partition Tolerance) not a realistic option for distributed systems?**
Because any system with more than one node communicating over a real network will eventually experience a partition — a dropped packet, a cut cable, a slow/unreachable process. You can't opt out of partitions happening, so partition tolerance is a given; the real decision is what to sacrifice (C or A) when one occurs. CA effectively only exists for single-node, non-distributed systems.

**4. Give two examples of CP-leaning systems and explain their design choice.**
ZooKeeper and etcd (and HBase) are CP: they refuse to serve requests they can't confirm are consistent with the rest of the cluster during a partition. This fits their use cases — leader election, configuration, and coordination — where two nodes disagreeing about the truth would cause serious bugs elsewhere in the system.

**5. Give two examples of AP-leaning systems and explain their design choice.**
Cassandra and DynamoDB are AP: they keep serving reads and writes on both sides of a partition, accepting that data may temporarily diverge, and reconcile conflicts afterward (e.g., via vector clocks or last-write-wins). This fits high-traffic, availability-critical use cases like shopping carts or social feeds, where an error page is worse than briefly stale data.

**6. What gap in CAP theorem does PACELC address?**
CAP says nothing about system behavior when there is no partition — which is the vast majority of the time for most systems. PACELC adds: "Else (no partition), choose between Latency and Consistency," capturing the everyday trade-off of how long to wait for replica acknowledgment before confirming a write or serving a read.

**7. Classify DynamoDB, Cassandra, and MongoDB (majority read/write concern) on the PACELC spectrum.**
- DynamoDB: PA/EL — favors availability during a partition, and favors low latency (eventual consistency) during normal operation, with an opt-in strongly consistent read mode.
- Cassandra: PA/EL by default, but tunable per query via consistency levels (ONE, QUORUM, ALL).
- MongoDB (majority concern): PC/EC — favors consistency both during a partition and during normal operation, at the cost of latency/availability.

**8. You're designing a global shopping cart service. Would you lean CP or AP, and why?**
AP. The cost of a customer seeing an error or being unable to add an item to their cart (lost sale, bad experience) is generally worse than the cost of occasionally reconciling a minor cart conflict later (e.g., an item that sold out being briefly shown as available). This mirrors Amazon's own original justification for building DynamoDB.

**9. You're designing the ledger for a banking system that transfers funds between accounts. Would you lean CP or AP, and why?**
CP. Allowing an inconsistent balance to be read or allowing a transfer to proceed without confirming funds across replicas risks double-spending or incorrect balances, which is a serious financial and regulatory problem. It's better to briefly reject or delay a transaction during a partition than to risk an incorrect, hard-to-reverse financial state.

**10. A teammate says "Cassandra is an AP database, so it can never be strongly consistent." Is this accurate?**
Not quite. Cassandra is tunable: while its defaults favor availability and low latency, you can set the consistency level to QUORUM or ALL on a per-query basis to get much stronger consistency guarantees, trading away some availability/latency. "AP" describes Cassandra's common posture and design center, not a hard limit on what it can do.

**11. What's a real-world, non-technical analogy for the difference between a CP and an AP system during a network split?**
A CP system is like a bank vault that requires two managers present to open — if one is unreachable, the vault stays locked no matter how inconvenient that is (favoring correctness). An AP system is like a chain of stores that keeps selling gift cards on a local paper ledger during an internet outage and reconciles balances with headquarters once the connection returns (favoring being open for business).

**12. Why do interviewers care about CAP/PACELC even though most engineers never explicitly configure a "CAP mode"?**
Because these trade-offs surface implicitly in nearly every distributed systems design decision — choosing a database, setting a write concern/consistency level, deciding how to handle a downstream service timeout, or designing retry/replication logic. Understanding CAP/PACELC gives you the vocabulary and mental model to reason about and justify those choices explicitly, rather than making them by accident.
