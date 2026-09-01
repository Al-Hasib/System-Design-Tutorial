# Study Notes: Logical Clocks & Time in Distributed Systems

## Definitions

- **Clock skew/drift:** The difference/divergence between independent machines' physical clocks, even under NTP synchronization.
- **NTP (Network Time Protocol):** Protocol for synchronizing physical clocks across machines against reference time servers — typically accurate to single-digit milliseconds under good conditions, but not guaranteed.
- **Lamport timestamp:** A per-process counter establishing a "happened-before" partial order: incremented on every local event, and set to `max(own, received) + 1` on receiving a message.
- **Happened-before relationship:** A partial order — if event A could have causally influenced event B, A "happened-before" B.
- **Vector clock:** An array of per-process counters; each process increments only its own slot locally, and takes an element-wise max (then increments its own slot) on receiving a message.
- **Concurrent events:** Two events where neither's vector clock dominates the other — neither could have causally influenced the other.

## Lamport Timestamps vs. Vector Clocks

| | Lamport Timestamps | Vector Clocks |
|---|---|---|
| Structure | Single counter per process | Array of counters, one per process |
| Guarantees | If A happened-before B, then timestamp(A) < timestamp(B) | Can determine happened-before AND detect true concurrency |
| Limitation | Cannot distinguish concurrent (unrelated) events from causally ordered ones | Higher overhead — size grows with number of processes |
| Typical use | Basic event ordering, simple distributed logging | Conflict detection in multi-replica systems (e.g., Dynamo, Riak) |

## Vector Clock Comparison Rules

Given vectors A and B:
- **A happened-before B** if every element of A ≤ corresponding element of B, and at least one is strictly less.
- **A and B are concurrent** if neither dominates the other (some elements higher in A, others higher in B) — this is the signal for a genuine conflict.

## When to Use Physical Time vs. Logical/Vector Clocks

| Use case | Right tool |
|---|---|
| Log timestamps for human debugging | Physical (NTP) time |
| Cache TTL / expiration windows | Physical (NTP) time — approximate precision is fine |
| Rate-limiting windows | Physical (NTP) time |
| Ordering causally-related events for correctness | Lamport timestamps |
| Detecting conflicting concurrent writes across replicas | Vector clocks |

## Key Numbers / Facts

- Leslie Lamport's "Time, Clocks, and the Ordering of Events in a Distributed System" (1978) introduced Lamport timestamps.
- NTP typically synchronizes clocks to within a few milliseconds under good network conditions, but this is not a hard guarantee — network issues can cause much larger skew.
- Amazon's Dynamo paper (2007) popularized vector clocks for conflict detection in eventually-consistent, multi-replica systems; the "shopping cart merge" example comes from this system.

## Summary

- Wall-clock time across machines cannot be trusted for correctness-critical event ordering, even with NTP.
- Lamport timestamps establish causal ("happened-before") order cheaply but can't tell true concurrency apart from arbitrary ordering.
- Vector clocks can detect true concurrency, which is exactly what's needed to know when two writes genuinely conflict and require explicit resolution rather than a silent, potentially wrong timestamp-based pick.
