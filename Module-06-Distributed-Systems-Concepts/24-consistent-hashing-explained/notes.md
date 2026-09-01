# Study Notes: Consistent Hashing

## Definitions

- **Consistent hashing**: A hashing scheme where nodes and keys are mapped into the same hash space, and adding or removing a node only requires remapping a small, proportional fraction of keys (roughly K/N), instead of remapping nearly the entire key space.
- **Hash ring**: The circular representation of the hash space (e.g., 0 to 2^32 - 1, wrapping from max back to 0) onto which node and key hash values are plotted. A key belongs to the first node encountered walking clockwise from the key's position.
- **Virtual nodes (vnodes)**: Multiple hash positions generated per physical node (e.g., by hashing `node_id#1`, `node_id#2`, ...), each placed independently on the hash ring. Used to even out load distribution and to allow weighting by node capacity.
- **Routing consistency**: The property that independent clients, given the same view of ring membership, compute the same node for the same key without coordination. Distinct from CAP-theorem "consistency" (data consistency across replicas).

## Why Naive `hash(key) mod N` Fails at Scale

- The modulus N is baked into every key's assignment. Changing N (adding/removing a node) changes almost every key's `mod N` result, because there is no structural relationship between mod-N and mod-(N+1) assignments.
- Worst case: up to `(N-1)/N` of all keys get remapped for a single node change (e.g., ~90% for a 10-node cluster).
- Consequences: mass cache misses, huge data-transfer/rebalancing cost, temporary overload on the origin/backing store, poor incremental scalability.

## Comparison: Naive Hashing vs Consistent Hashing

| Aspect | Naive `mod N` Hashing | Consistent Hashing (Ring) |
|---|---|---|
| Keys remapped on resize | Up to (N-1)/N of all keys (~90% for N=10) | Roughly K/N keys (only the adjacent arc) |
| Rebalancing cost | High — near full data shuffle | Low — proportional to size of change |
| Implementation complexity | Very simple (single modulo op) | Moderate (ring structure, sorted positions, lookup by successor) |
| Load balance across nodes | Even, by construction, for fixed N | Uneven with few points per node; needs virtual nodes to even out |
| Handles heterogeneous hardware | No (needs custom logic) | Yes — vary virtual node count per node |
| Coordination needed | Needs global agreement on N | Needs eventually-consistent view of ring membership (e.g., gossip) |
| Typical use cases | Small/fixed-size clusters, simple in-memory partitioning | Distributed caches, NoSQL stores (DynamoDB, Cassandra, Riak), CDN/load-balancer routing |

## Key Numbers to Remember

- Naive resize disruption: up to `(N-1)/N` of keys move (e.g., 9/10 = 90% for N=10).
- Consistent hashing resize disruption: approximately `K/N` keys move (K = total keys, N = node count).
- Typical virtual nodes per physical node in production systems: roughly 100-200 (some systems configure more or fewer depending on cluster size and desired balance).
- Replication factor (R) is layered on top of the ring: writes/reads typically touch the next R distinct physical nodes walking clockwise from a key's position (common defaults: R = 3 in Dynamo-style systems).

## Interview Revision — Bullet Summary

- Naive mod-N hashing breaks on resize because the modulus changes; consistent hashing fixes this by using a ring and "next node clockwise" ownership.
- Only the arc adjacent to the topology change is affected on node add/remove — approximately K/N keys move, not nearly all of them.
- Virtual nodes: multiple ring positions per physical node; solves (1) load imbalance from random placement, (2) heterogeneous capacity weighting, (3) concentrates failover load across many nodes instead of one neighbor.
- "Consistent" = routing consistency (same key, same node, across independent observers), not CAP-style strong consistency.
- Real systems layer replication (N replicas), conflict resolution (vector clocks, LWW, read-repair), and membership propagation (gossip) on top of the basic ring.
- Real-world users: DynamoDB, Cassandra (token ring + vnodes), Riak, CDNs, and load balancers/service meshes needing sticky, resize-tolerant routing.
