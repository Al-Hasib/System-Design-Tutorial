# Study Notes: CAP Theorem & PACELC

## Definitions

- **Consistency (C)**: Every read returns the most recent write (or an error). All nodes see the same data at the same time, as if there were a single copy.
- **Availability (A)**: Every request to a non-failed node receives a (non-error) response, without guarantee that it contains the most recent write.
- **Partition Tolerance (P)**: The system continues to operate despite an arbitrary number of messages being dropped or delayed by the network between nodes.
- **PACELC**: Extension of CAP — *if Partition, choose Availability or Consistency; Else (no partition, normal operation), choose Latency or Consistency.*

## CAP Theorem Trade-offs

| Aspect | CP (Consistency + Partition Tolerance) | AP (Availability + Partition Tolerance) |
|---|---|---|
| Behavior during partition | Rejects/blocks requests on the minority/unreachable side rather than risk stale reads or lost writes | Continues to serve reads/writes on both sides, accepting possible staleness or conflicts |
| Consistency guarantee | Strong (linearizable/majority-based) | Eventual (resolved later via reconciliation) |
| Example databases | ZooKeeper, etcd, HBase, MongoDB (majority read/write concern) | Cassandra, DynamoDB, Riak, CouchDB |
| Typical use cases | Leader election, config/coordination services, financial ledgers, inventory counts that must not oversell | Shopping carts, social feeds, session stores, analytics/telemetry, product catalogs |

Note: "C+A without P" is not a real option for a multi-node system communicating over a network — partitions are a fact of life, so CAP in practice is a CP vs. AP decision.

## PACELC Table

| System | P behavior (during partition) | E behavior (normal operation) |
|---|---|---|
| DynamoDB | PA — favors Availability | EL — favors Latency (eventually consistent reads by default; strongly consistent reads optional at higher latency) |
| Cassandra | PA — favors Availability (tunable) | EL — favors Latency by default (tunable consistency level per query: ONE/QUORUM/ALL) |
| MongoDB (majority read/write concern) | PC — favors Consistency (primary steps down if it can't reach a majority) | EC — favors Consistency (waits for majority acknowledgment) |
| ZooKeeper / etcd | PC — favors Consistency | EC — favors Consistency (linearizable reads/writes by design) |
| HBase | PC — favors Consistency | EC — favors Consistency (single active region server per region) |

## Key Nuances

- CAP theorem describes behavior **only during an active network partition** — it says nothing about trade-offs during normal, healthy-network operation.
- Partition tolerance is not optional in a real distributed system; it's a given. The meaningful design choice is what to sacrifice — Consistency or Availability — *when* a partition occurs.
- A system is not permanently "CP" or "AP" as an identity — it's a design choice describing what it prioritizes at the moment of a partition; many systems are tunable per-request (e.g., Cassandra consistency levels).
- PACELC is more complete than CAP because it also captures the constant, everyday latency-vs-consistency trade-off that exists even when there's no partition at all.
- The "right" trade-off is a business/domain decision: compare the cost of incorrect/stale data vs. the cost of downtime or slow responses.
- Many "AP" systems still offer stronger consistency options on demand (e.g., DynamoDB strongly consistent reads, Cassandra QUORUM) — the label describes the default/common posture, not an absolute limit.
