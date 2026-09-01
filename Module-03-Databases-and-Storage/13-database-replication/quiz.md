# Practice & Interview Questions

1. **What is database replication, and what two main problems does it solve?**
   Replication is copying and maintaining the same data across multiple database nodes. It solves (1) availability/durability — a failed node doesn't mean lost data or downtime, since another node has a copy — and (2) read scaling — read traffic can be spread across multiple nodes instead of one.

2. **What is the difference between synchronous and asynchronous replication?**
   Synchronous replication waits for the replica to confirm it has received a write before acknowledging the client, favoring durability at the cost of latency. Asynchronous replication acknowledges the client immediately and replicates in the background, favoring speed but creating a window where data on the primary hasn't reached replicas yet.

3. **What is replication lag, and why does it usually exist?**
   Replication lag is the delay between a write landing on the primary/master and that write becoming visible on a follower/replica. It exists primarily because most systems use asynchronous replication for performance, so replicas apply changes slightly after the primary does.

4. **Scenario: A user posts a comment, gets redirected to the comments page, and their own comment is missing. What's happening and how do you fix it?**
   This is the classic read-your-writes problem: the write went to the master, but the page's read was served from a replica that hasn't caught up yet (replication lag). Fixes include routing that user's immediate follow-up reads to the master for a short window, tracking a replication position/token and waiting for the replica to catch up to it, or using session affinity to the master after a write.

5. **In Master-Slave replication, why can reads scale so much more easily than writes?**
   Because you can add as many follower/replica nodes as you want, each capable of independently serving read traffic, whereas all writes must go through the single master — adding more masters isn't possible in this topology, so write throughput is capped by what one machine can handle.

6. **What is failover, and what makes it risky?**
   Failover is detecting that a master/primary has failed and promoting a replica to take its place. It's risky because there's a detection and promotion delay (writes are unavailable during that window), and if replication was asynchronous, any writes that hadn't yet reached the promoted replica can be lost.

7. **What problem does Master-Master (multi-leader) replication solve that Master-Slave cannot?**
   It allows multiple nodes to accept writes directly and independently, enabling write scaling across nodes and multi-region active-active setups where users write to a geographically close leader with low latency and continued write availability even if one region's leader fails.

8. **What is a write conflict in multi-leader replication, and name two strategies to resolve it.**
   A write conflict occurs when two leaders each accept a write to the same record at roughly the same time, producing two disagreeing values once they replicate to each other. Two resolution strategies: last-write-wins (LWW), which keeps whichever write has the later timestamp, and vector clocks, which track causal history to detect true concurrency versus one write causally following another (sometimes surfacing genuine conflicts to the application).

9. **What is a downside of last-write-wins (LWW) conflict resolution?**
   It can silently discard a legitimate, intentional write just because it happened to have an earlier timestamp — including cases caused by slightly unsynchronized clocks across nodes — with no opportunity for the application or user to reconcile the two writes.

10. **Scenario: You're designing a read-heavy blog platform with a 1000:1 read-to-write ratio. Which replication topology would you choose, and why?**
    Master-Slave (leader-follower). Since writes are rare and reads dominate, a single master easily handles the write volume while adding read replicas behind a load balancer scales read capacity linearly — there's no need for the added conflict-resolution complexity of master-master.

11. **Scenario: You're building a globally distributed collaborative app where users in the US, Europe, and Asia all need low-latency writes and continued availability during regional outages. Which topology fits, and what must you also design for?**
    Master-Master (multi-leader), with a master per region so users write to their nearest region with low latency and the app stays available for writes even if one region goes down. You must also design a conflict resolution strategy (e.g., vector clocks or app-level merge logic) since concurrent writes to the same data across regions are expected.

12. **Compare Master-Slave and Master-Master on complexity and explain why.**
    Master-Slave is simpler: there's a single writer, so there's never a need to detect or resolve conflicting writes, and the only real operational complexity is the failover process. Master-Master is more complex because it requires bidirectional replication between leaders plus an active conflict detection/resolution strategy, since multiple nodes can accept conflicting writes concurrently.
