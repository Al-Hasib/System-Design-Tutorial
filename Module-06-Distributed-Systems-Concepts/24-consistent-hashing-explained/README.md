# Consistent Hashing Explained

Difficulty: Advanced | Estimated length: 15-20 min | Prerequisites: [Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md), [CAP Theorem and PACELC](../../Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md)

## Learning Objectives

- Explain why naive `hash(key) mod N` partitioning collapses under cluster resizing.
- Describe the hash ring model and how keys and nodes are placed on it.
- Explain how virtual nodes solve load imbalance and heterogeneous capacity problems.
- Reason quantitatively about how many keys move when a node joins or leaves the ring.
- Connect consistent hashing to real systems: DynamoDB, Cassandra, Riak, and CDN/load-balancer routing.

## Script

### Hook / Intro

Picture this: you're running a 10-node Memcached cluster, it's been humming along fine for months, and then traffic grows, so you add a couple of nodes. The moment you do, your cache hit rate falls off a cliff. Every client suddenly starts asking different servers for the same keys, your origin database gets hammered, and you spend the afternoon explaining to your team why "just adding capacity" caused an outage instead of preventing one.

That's the naive-hashing failure mode, and it's exactly what consistent hashing was invented to fix. You already know sharding and partitioning, you already know CAP theorem trade-offs — today we're going one level deeper into the actual mechanism that lets systems like DynamoDB, Cassandra, and every major CDN add and remove nodes without a full data reshuffle. Let's get into it.

### The Problem with Naive Hashing (mod N)

The obvious way to distribute keys across N nodes is `node = hash(key) mod N`. It's simple, it gives a uniform distribution for a good hash function, and for a fixed cluster size it works fine. The problem is what happens the moment N changes.

If you go from N to N+1 nodes, the modulus changes, and that changes the assignment for almost every key. A key that was at `hash(key) mod 10 = 7` might now land at `hash(key) mod 11 = 3`. There's no structural relationship between mod-N and mod-(N+1) assignments — it's effectively a fresh random remapping. In the worst case, close to `(N-1)/N` of all keys move. For a 10-node cluster, that's roughly 90% of your data getting reshuffled for a one-node change. For a cache, that means a near-total cold start. For a sharded database, that means moving nearly the entire dataset across the network just to add capacity — which defeats the purpose of scaling incrementally.

This is the core problem consistent hashing solves: we want adding or removing a node to only affect the keys that node actually owns, and nothing else.

### The Hash Ring

The trick is to stop thinking of hashing as "pick a bucket index" and start thinking of it as "pick a point in space." Consistent hashing maps both nodes and keys into the same hash space — typically a fixed range like 0 to 2^32 minus 1 — and treats that space as a ring: after the maximum value, you wrap back around to zero. Picture a clock face, except instead of 12 hours you have billions of positions.

Each node is hashed — often using something like the node's IP and port — and placed at the resulting position on the ring. Each key is hashed the same way and also placed on the ring. To find which node owns a key, you walk clockwise from the key's position until you hit the first node. That node owns the key. Every node effectively owns the arc of the ring between itself and the previous node going counter-clockwise.

Now here's why this fixes the rebalancing problem. When you add a new node, it lands at some position on the ring, and it only "steals" the keys in the arc immediately counter-clockwise of it — keys that used to belong to the next node clockwise. Every other node's ownership is completely undisturbed. Symmetrically, when a node is removed, only its keys need to move, and they all go to the next node clockwise — everyone else keeps what they had. Instead of remapping close to 100% of keys, you're remapping roughly K/N keys, where K is the total key count and N is the number of nodes. Adding a node to a 10-node ring moves around 1/11th of the data, not 90% of it. That's the entire value proposition in one sentence: minimal disruption, proportional to the size of the change, not the size of the cluster.

It's also worth being precise about what "consistent" means here — it's not about strong consistency in the CAP sense, it's about consistency of routing: independent clients, without talking to each other, will compute the same node for the same key as long as they have the same view of ring membership. That property is what makes the ring usable as a decentralized routing mechanism instead of requiring a central lookup service.

### Virtual Nodes and Load Balancing

There's a catch with the naive ring: if you only place each physical node once, the arcs are randomly sized. With a small number of nodes, random placement on a large hash space can produce wildly uneven arc lengths — one node might own 40% of the keyspace while another owns 5%. You also can't easily give a beefier machine more load than a smaller one.

The fix is virtual nodes, sometimes called "vnodes" or replicas in the ring. Instead of hashing a physical node once, you hash it multiple times with different suffixes — node-A#1, node-A#2, node-A#3, and so on — and place all of those points on the ring. Each physical node now owns many small, scattered arcs instead of one large contiguous one. With enough virtual nodes per physical node — production systems commonly use somewhere in the range of 100 to 200, though some go higher — the law of large numbers takes over and the total keyspace owned by each physical node converges toward an even share, statistically, regardless of where the hash function happens to place things.

Virtual nodes also solve the heterogeneous-hardware problem directly: if one machine has twice the RAM or disk of another, you just give it twice as many virtual nodes. It ends up owning roughly twice the arc length and therefore roughly twice the data and traffic — all without any special-casing in the routing logic itself.

And virtual nodes actually improve the failure-recovery story too, which we'll get to next: when a node fails, instead of all its load being dumped on exactly one neighbor, it's spread across many different physical nodes, because each of the failed node's virtual points has a different clockwise neighbor.

### Handling Node Failure and Growth

Let's walk through the two operations concretely.

Node addition: the new node computes its set of virtual node positions and inserts them into the ring. For each virtual point, it takes ownership of the arc immediately before it, which previously belonged to whatever node was next clockwise. Data for those specific key ranges streams from the old owner to the new node. No other node in the cluster needs to do anything. Read and write routing updates as soon as clients or a coordinator refresh their view of ring membership.

Node removal, whether planned decommissioning or an actual failure, is the mirror image: that node's virtual points are removed from the ring, and its arcs merge into whatever node is now the next clockwise neighbor for each point. Because virtual nodes scatter a physical node's ownership across many neighbors, that failover load is distributed across the cluster instead of concentrated on one unlucky neighbor.

In real systems this is layered with replication for availability: DynamoDB and Cassandra-style systems don't put a single copy of a key on the ring's immediate successor — they replicate to the next R distinct physical nodes walking clockwise, so a single node failure doesn't mean data loss, it just means falling back to replicas while the ring heals. This is also where the CAP trade-offs you already know come back into play: how many replicas must acknowledge a write, how conflicts between replicas get resolved during a partition — vector clocks, last-write-wins, read-repair — all sit on top of this ring-based placement layer.

One more subtlety: ring membership itself needs to be agreed upon, or at least eventually converge, across nodes and clients. Systems like Cassandra use gossip protocols so every node eventually learns the current ring topology; DynamoDB-style systems similarly propagate membership changes so routing stays consistent cluster-wide, even if there's a brief window of disagreement during a topology change.

### Real-World Example

You'll find consistent hashing under the hood of most large-scale distributed data stores and routing layers. Amazon's Dynamo paper, which we'll link in the resources, is the system that really popularized this technique for databases — it uses a hash ring with virtual nodes for partitioning, and DynamoDB inherited that lineage. Apache Cassandra uses the same idea explicitly: its partitioner maps row keys onto a token ring, and virtual nodes (vnodes) are a standard configuration for spreading load evenly and speeding up bootstrap and repair. Riak, another Dynamo-inspired store, uses a nearly identical consistent-hashing ring with vnodes for its partitioning scheme.

It's not just databases, either. CDNs and load balancers use consistent hashing to route requests to origin servers or cache nodes so that the same request key — a URL or a client identifier — consistently lands on the same backend, which maximizes cache locality, and so that adding or removing an edge or backend node doesn't invalidate the entire cache's worth of routing decisions at once. Some client-side load balancing libraries and service meshes use consistent hashing for exactly this reason when they need sticky routing that also tolerates a changing pool of backends.

### Recap

Let's tie it together. Naive mod-N hashing remaps nearly everything when the cluster size changes, because the modulus itself changes. Consistent hashing fixes that by mapping nodes and keys onto a shared ring and assigning each key to the next node clockwise, so only the arc near a topology change is affected — roughly K/N keys move instead of nearly all of them. Virtual nodes solve the resulting load-imbalance problem by giving each physical node many small scattered arcs instead of one large one, which also lets you weight capacity and spreads failover load across the cluster. And this is a routing-consistency property, not a CAP-consistency property — it's what lets independent clients agree on where a key lives without a central coordinator. This is the mechanism underneath DynamoDB, Cassandra, Riak, and a huge swath of CDN and load-balancer routing.

### What's Next

Now that you know how distributed systems decide *where* a request or key goes, the natural next question is how they decide *whether* to let a request through at all under load. In the next video, we're covering rate limiting algorithms — token bucket, leaky bucket, fixed and sliding window counters — and how systems combine them with the kind of distributed routing you just learned to enforce limits consistently across a fleet of nodes. See you there.

## Key Takeaways

- Naive `hash(key) mod N` remaps up to `(N-1)/N` of all keys on any cluster resize, because the modulus itself changes.
- Consistent hashing maps nodes and keys onto a shared hash ring; a key belongs to the first node found walking clockwise from its position.
- Adding or removing a node only affects the adjacent arc, moving roughly K/N keys instead of nearly all of them.
- Virtual nodes give each physical node many scattered arcs, producing even load distribution and letting you weight nodes by capacity.
- "Consistent" here refers to routing consistency (independent clients agreeing on key ownership), not CAP-style data consistency.
- Production systems typically use around 100-200 virtual nodes per physical node, layered with replication (N replicas clockwise) for availability.
- DynamoDB, Cassandra, and Riak all use ring-based consistent hashing with virtual nodes; CDNs and load balancers use it for sticky, resize-tolerant routing.
