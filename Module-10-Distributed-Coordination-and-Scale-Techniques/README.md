# Module 10: Distributed Coordination & Scale Techniques

Module 6 covered the big-name distributed systems problems — consensus, distributed transactions, consistency models. This module fills in three more tools that come up constantly once you're actually building at scale, but rarely get their own dedicated treatment: how multiple independent processes safely share a single resource without a shared memory space to synchronize through, how a distributed system agrees on the *order* events happened in when there's no single shared clock to rely on, and how to answer approximate questions ("have I seen this before?", "roughly how many unique items?") over massive datasets without the memory cost of an exact answer. None of these are exotic — they show up in real production systems constantly, and in interviews whenever the scale of a problem makes an exact, naive solution obviously too expensive.

## Videos in This Module

| # | Title | Description | Link |
|---|-------|-------------|------|
| 40 | Distributed Locking: Redlock, ZooKeeper & etcd | How multiple independent processes safely coordinate exclusive access to a shared resource — and why a single-node lock (like a database row lock) isn't enough once you're distributed. | [40-distributed-locking-redlock-zookeeper-and-etcd](40-distributed-locking-redlock-zookeeper-and-etcd/README.md) |
| 41 | Logical Clocks & Time in Distributed Systems | Why you can't trust wall-clock time to order events across machines, and how Lamport timestamps and vector clocks establish a reliable "happened-before" relationship instead. | [41-logical-clocks-and-time-in-distributed-systems](41-logical-clocks-and-time-in-distributed-systems/README.md) |
| 42 | Probabilistic Data Structures: Bloom Filters, HyperLogLog & Count-Min Sketch | Answering "have I seen this?", "roughly how many uniques?", and "roughly how often?" over huge datasets using a tiny, fixed amount of memory. | [42-probabilistic-data-structures-bloom-filters-hyperloglog-and-count-min-sketch](42-probabilistic-data-structures-bloom-filters-hyperloglog-and-count-min-sketch/README.md) |
