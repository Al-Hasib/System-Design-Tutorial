# Why This Topic Matters: Consistent Hashing

> **In one sentence:** The obvious way to spread keys across N servers — `hash(key) % N` — remaps almost every key the moment N changes, which turns "add one cache node" into "invalidate the entire cache at peak traffic."

## The World Before This Idea

You have four cache servers. You route keys with `hash(key) % 4`. It distributes evenly and it's one line of code.

Then you add a fifth server. Now keys route by `hash(key) % 5`. Consider what that means: a key that hashed to 7 used to go to server 3 (`7 % 4`), and now goes to server 2 (`7 % 5`). Work through the arithmetic across all keys and roughly **80% of them move**. Every one of those is now a cache miss.

So at the exact moment you added capacity because you were under load, you invalidate 80% of your cache and drop that traffic onto the database. The scaling operation causes the outage. The same thing happens — worse, unplanned — when a server *dies*.

## The Problems It Solves

### 1. Mass remapping on every topology change
**What you see:** Adding or removing a node causes a cache-miss storm and a database overload.

**Why it happens:** Modulo hashing ties every key's destination to the *total count* of nodes. Change the count, change (almost) every mapping.

**How consistent hashing solves it:** Keys and nodes are both placed on a hash ring. A key belongs to the first node clockwise from it. Adding a node only captures the keys between it and its predecessor — on average **K/N keys**, where K is the key count and N the node count. Add a fifth node to a four-node ring and roughly 20% of keys move instead of 80%. Remove a node and only *its* keys move; everyone else is untouched.

### 2. Scaling that's too dangerous to do
**What you see:** Teams avoid adding cache or database nodes because the remapping cost is worse than the capacity problem. Capacity planning becomes "provision for peak forever."

**Why it happens:** If scaling causes an incident, you stop scaling.

**How consistent hashing solves it:** It makes elasticity safe. Nodes can be added and removed routinely — for autoscaling, rolling upgrades, or instance replacement — because the blast radius is bounded and small.

### 3. Uneven distribution and hotspots
**What you see:** With few nodes placed randomly on a ring, one node ends up owning a huge arc and takes far more than its share. Worse, when it fails, *all* of its load lands on exactly one neighbor, which then also fails — a cascading failure.

**Why it happens:** A small number of random points on a circle produces very uneven gaps.

**How virtual nodes solve it:** Each physical node is placed at many points on the ring (100-200 is typical). Distribution smooths out dramatically, and when a node fails its many small arcs are inherited by many different neighbors, spreading the load instead of concentrating it. Virtual nodes also let you weight heterogeneous hardware — a machine with twice the memory gets twice the ring positions.

### 4. Stateful services that can't be load-balanced naively
**What you see:** You need a specific user's data, session, or WebSocket connection to consistently reach the node that holds it, but round-robin sends each request somewhere different.

**Why it happens:** Standard load balancing assumes interchangeable backends. Stateful ones aren't interchangeable.

**How consistent hashing solves it:** It provides a stable, deterministic key-to-node mapping that every client can compute independently — no coordinator, no lookup table — and that stays stable as the fleet changes.

## The Price You Pay

- **More complexity than one line of modulo.** A ring structure, virtual node bookkeeping, and a way for all clients to agree on the current membership.
- **Membership must be distributed and agreed.** Every client needs a consistent view of which nodes are on the ring. If two clients disagree, they route the same key differently. Keeping that view current usually means a gossip protocol or a coordination service (ZooKeeper, etcd) — which is itself a distributed systems problem.
- **Hotspots still exist at the key level.** Consistent hashing evens out *key distribution*; it does nothing about one key being requested a million times a second. A celebrity user or a viral item still overwhelms one node, and needs a separate remedy (key splitting, local caching of hot keys, or dedicated handling).
- **Rebalancing still moves data.** K/N is much better than most-of-everything, but for a large dataset it's still real network traffic and real time, and it happens while serving live requests.

## When You Need It — and When You Don't

| Use it when | Skip it when |
|---|---|
| Keys map to nodes and the node set changes | The node set is fixed and never changes |
| You're distributing a cache, shard set, or partition space | The backends are stateless and interchangeable |
| Nodes fail or autoscale routinely | A central coordinator maintaining a lookup table is acceptable |
| Clients must route without a central lookup | Data volume is small enough that full remapping is cheap |

## Why This Shows Up in Interviews

Consistent hashing is a classic interview topic because it's a specific, teachable idea with a clear before-and-after, and because it appears in the internals of systems everyone claims to know: Cassandra, DynamoDB, Riak, memcached clients, Envoy's ring-hash balancer, and most CDN request routing. Expect to be asked why modulo hashing fails, to walk through the ring, and to explain virtual nodes — the last of which is the part candidates most often miss, and the part that matters most in practice.

## How It Connects

Consistent hashing is the mechanism that makes **sharding** (topic 14) and **distributed caching** (topic 19) operationally survivable, and it's what lets a **load balancer** (topic 7) route stably to stateful backends. It presupposes the cluster membership problem that **consensus algorithms** (topic 27) and **coordination services** (topic 40) solve. You'll see it again in nearly every large-scale case study in Module 12.

**Next:** [Rate Limiting Algorithms](../25-rate-limiting-algorithms/why.md) — controlling how much load reaches those nodes in the first place.
