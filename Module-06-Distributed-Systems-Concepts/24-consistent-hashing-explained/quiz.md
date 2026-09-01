# Practice & Interview Questions

**1. Why does naive `hash(key) mod N` partitioning perform poorly when a cluster is resized?**
Because the modulus N is part of every key's assignment formula, changing N (adding/removing a node) changes the `mod N` result for almost every key, with no structural relationship between the old and new assignments. In the worst case, up to `(N-1)/N` of all keys get remapped for a single node change, causing mass cache misses or a near-total data reshuffle.

**2. Describe the hash ring model in your own words.**
- Nodes and keys are both hashed into the same fixed range (e.g., 0 to 2^32-1), treated as a circle that wraps from the maximum value back to zero.
- Each node occupies a position on the ring based on its hash.
- A key belongs to the first node encountered walking clockwise from the key's own hashed position.

**3. You have a cache cluster of 5 nodes using consistent hashing and add a 6th node. Roughly how many keys move, and why?**
Roughly K/6 of the keys move (where K is total key count) — only the arc of the ring that the new node inserts itself into is affected, and those keys come from exactly one existing neighbor (the node that used to own that arc). Every other node's keys are untouched, unlike naive mod-N hashing, which would remap the majority of keys.

**4. What problem do virtual nodes solve, and how?**
With only one hash position per physical node, random placement on the ring can produce very uneven arc lengths, so some nodes end up owning far more of the keyspace than others. Virtual nodes hash each physical node multiple times (commonly ~100-200 positions), scattering many small arcs per node around the ring; by the law of large numbers, each physical node's total owned keyspace converges toward an even share.

**5. How do virtual nodes help with heterogeneous hardware capacity?**
You can assign a more powerful machine a larger number of virtual nodes (e.g., double the vnodes for double the RAM/disk), so it ends up owning proportionally more of the ring's arcs and therefore proportionally more data and traffic — without changing the routing algorithm itself, just the vnode count configuration per physical node.

**6. What happens to the ring, and to data, when a node fails unexpectedly?**
The failed node's virtual points are removed (or treated as unreachable) from the ring, and each of its arcs is taken over by the next node clockwise from that virtual point. Because virtual nodes scatter a physical node's ownership across many different neighbors, the failed node's load is spread across multiple physical nodes rather than dumped entirely on one neighbor. In replicated systems, reads/writes fail over to existing replicas while the ring heals.

**7. Is "consistent" in consistent hashing the same as "consistency" in the CAP theorem? Explain.**
No. Consistent hashing's "consistency" refers to routing consistency: independent clients or nodes, given the same view of ring membership, compute the same owner for the same key without needing to coordinate. CAP's consistency refers to whether all replicas of a piece of data reflect the same, most recent value. A system can have perfectly consistent routing via a hash ring while still being an AP (eventually consistent) system in the CAP sense — Dynamo and Cassandra are the classic examples.

**8. Why do production systems typically use replication on top of the hash ring instead of storing each key on a single node?**
A single copy per key means any single node failure causes data loss or unavailability for that key's arc. Systems like Dynamo, DynamoDB, and Cassandra replicate each key to the next R distinct physical nodes walking clockwise from its position (commonly R=3), so a node failure just means falling back to replicas rather than losing data, at the cost of needing conflict resolution (vector clocks, last-write-wins, read-repair) between replicas.

**9. How does a hash ring stay synchronized across all nodes and clients as the topology changes?**
Ring membership needs to be propagated so that routing decisions converge across the cluster. Systems like Cassandra use gossip protocols, where nodes periodically exchange membership state with random peers until everyone eventually agrees on the current ring topology; there may be a brief window of disagreement during a topology change, which replication and read-repair help paper over.

**10. Compare the rebalancing cost of naive mod-N hashing versus consistent hashing when scaling a 10-node cluster to 11 nodes.**
- Naive mod-N: up to (10)/(11) ≈ 91% of keys can be remapped, since the modulus itself changes for nearly all keys.
- Consistent hashing: only about 1/11 (~9%) of keys move — just the arc the new node inserts itself into, taken from a single existing neighbor.
- This order-of-magnitude difference is the entire reason consistent hashing is preferred for systems that need to scale incrementally without large data-transfer costs.

**11. Give two real-world categories of systems that use consistent hashing, and what problem it solves for each.**
- Distributed NoSQL data stores (DynamoDB, Cassandra, Riak): partitioning data across storage nodes so the cluster can grow/shrink without moving nearly all data, while using vnodes to balance load across heterogeneous or uneven hardware.
- CDNs and load balancers: routing requests/cache lookups by key (URL, client ID) so the same key consistently lands on the same backend for cache locality, while tolerating backend pool changes without invalidating the entire routing/cache state at once.

**12. Suppose your monitoring shows one node in a consistent-hashing ring is consistently handling twice the traffic of its peers, even though all physical machines are identical. What is the likely cause and fix?**
The likely cause is an insufficient number of virtual nodes (or an unlucky hash distribution) causing that physical node's total owned arc length to be disproportionately large. The fix is to increase the virtual node count per physical node (moving toward the ~100-200+ range) so the law of large numbers evens out the arc-length distribution across nodes; if the hash function itself is poor quality, switching to a better-distributed hash function can also help.
