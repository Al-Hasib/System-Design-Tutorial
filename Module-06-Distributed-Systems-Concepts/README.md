# Module 6: Distributed Systems Concepts (Advanced)

This module is the advanced core of the course. Having covered databases, caching, and messaging, we now tackle the hard problems that emerge the moment a system runs across many machines instead of one: how to distribute load fairly, how to protect services from overload and cascading failure, how to get independent nodes to agree on a single truth, and how to reason about consistency when data lives in more than one place at once. These are the concepts that separate "I can use a database" from "I can design a distributed system," and they show up constantly in senior-level system design interviews.

| # | Title | Description | Link |
|---|-------|-------------|------|
| 24 | Consistent Hashing Explained | How to distribute keys across nodes so that adding/removing servers reshuffles as little data as possible | [24-consistent-hashing-explained](./24-consistent-hashing-explained/README.md) |
| 25 | Rate Limiting Algorithms (Token Bucket, Leaky Bucket, Sliding Window) | Comparing the major algorithms used to throttle traffic and protect services | [25-rate-limiting-algorithms](./25-rate-limiting-algorithms/README.md) |
| 26 | Circuit Breaker, Retry & Bulkhead Patterns | Resilience patterns that stop failures in one service from cascading through a whole system | [26-circuit-breaker-retry-and-bulkhead-patterns](./26-circuit-breaker-retry-and-bulkhead-patterns/README.md) |
| 27 | Consensus Algorithms: Paxos & Raft | How distributed nodes agree on a single value or leader despite failures | [27-consensus-algorithms-paxos-and-raft](./27-consensus-algorithms-paxos-and-raft/README.md) |
| 28 | Distributed Transactions: Two-Phase Commit & Saga Pattern | Coordinating atomic-like operations across multiple services or databases | [28-distributed-transactions-2pc-and-saga](./28-distributed-transactions-2pc-and-saga/README.md) |
| 29 | Data Consistency Models & Idempotency in Distributed Systems | Strong vs eventual vs causal consistency, and how idempotency keeps retries safe | [29-data-consistency-models-and-idempotency](./29-data-consistency-models-and-idempotency/README.md) |
