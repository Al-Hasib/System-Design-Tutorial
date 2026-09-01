# Study Notes: Data Consistency Models & Idempotency

## Definitions

- **Consistency model**: A contract specifying what values/orderings a client is allowed to observe when reading data that may be replicated or concurrently written.
- **Linearizability (strong consistency)**: Every operation appears to occur instantaneously at some point between its start and end; all clients observe the same total order of operations, matching real time.
- **Eventual consistency**: If no new writes occur, all replicas will eventually converge to the same value; no guarantee on time-to-converge or intermediate ordering.
- **Causal consistency**: Operations that are causally related (happens-before, e.g., a reply to a comment) are seen in the same order by all nodes; unrelated (concurrent) operations may be seen in different orders on different nodes.
- **Read-your-writes**: A client always sees its own prior writes in subsequent reads.
- **Monotonic reads**: A client never sees data "go backward in time" across successive reads.
- **Monotonic writes**: A client's writes are applied in the order it issued them.
- **Session consistency**: A bundle of the above client-centric guarantees, scoped to one client's session.
- **Idempotency**: An operation is idempotent if performing it N times has the same effect as performing it once.
- **Idempotency key**: A client-generated unique ID for a logical operation, used by the server to detect and safely handle retries of the same request.

## Consistency Model Comparison

| Model | Guarantee | Availability/Latency Cost | Example Systems |
|---|---|---|---|
| Linearizable (strong) | Total real-time order; reads always see latest write | High — needs leader/quorum/consensus; unavailable to minority side during partition | ZooKeeper, etcd, Google Spanner |
| Causal | Happens-before order preserved; concurrent ops may reorder | Medium — needs causal metadata (vector clocks) but no global lock-step | COPS, some Cosmos DB configurations |
| Read-your-writes / Monotonic reads | Per-client session guarantees only | Low-medium — sticky sessions or version tracking per client | Many social/consumer apps |
| Eventual | Convergence only, no ordering/time guarantee | Lowest — all replicas independently available | DNS, Cassandra (default), DynamoDB (eventual mode), S3 (historically) |

## Idempotent vs Non-Idempotent Operations

| Operation | Idempotent? | Why |
|---|---|---|
| `PUT /user/5 {name: "Sam"}` | Yes | Setting the same state repeatedly has the same result |
| `DELETE /user/5` | Yes | Deleting an already-deleted resource is a no-op |
| `POST /orders` (create new order) | No | Each call creates a new resource — retries create duplicates |
| `POST /accounts/5/increment-balance` | No | Repeating adds the amount multiple times |
| `POST /payments` with idempotency key | Made idempotent | Server deduplicates by key, safe to retry |

## Idempotency Key Implementation Approach

- Client generates a UUID per logical operation (not per network attempt) and sends it in a header (e.g., `Idempotency-Key`).
- Server, on first sight of the key, processes the request and stores `{key -> result}` (with a TTL, e.g., 24 hours) in a fast store (Redis, or a DB table with a unique constraint on the key).
- On a retry with the same key, the server short-circuits: returns the stored result without reprocessing.
- Use atomic "check-and-set" (unique constraint / conditional write) to avoid race conditions if the same key arrives concurrently.

## Interview Revision Bullets

- Consistency model choice is a direct consequence of CAP/PACELC trade-offs made at the replication layer.
- Linearizability is the strongest and most expensive guarantee; it requires coordination (leader election, consensus, or quorum reads/writes).
- Eventual consistency maximizes availability/latency but pushes conflict resolution (LWW, vector clocks, CRDTs) onto the application or storage engine.
- Causal consistency plus session guarantees (read-your-writes, monotonic reads) is a common practical middle ground for consumer applications.
- Idempotency is orthogonal to consistency: it's about making retries safe, not about how fresh data is — but the two interact constantly in distributed systems (e.g., retrying a write under eventual consistency).
- HTTP semantics: GET, PUT, DELETE are idempotent by spec; POST is not — which is why mutating POST endpoints that involve money or side effects need explicit idempotency keys.
