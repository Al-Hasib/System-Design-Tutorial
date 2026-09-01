# Diagrams: Data Consistency Models & Idempotency

The consistency spectrum, from strongest to weakest guarantee.

```mermaid
flowchart LR
    A["Linearizable\n(Strong)\nSingle total order,\nhigh coordination cost"] --> B["Causal Consistency\nHappens-before order\npreserved"]
    B --> C["Session Guarantees\nRead-your-writes,\nMonotonic reads/writes"]
    C --> D["Eventual Consistency\nConverges eventually,\nno ordering guarantee"]

    style A fill:#f96,stroke:#333
    style D fill:#9cf,stroke:#333
```
*Caption: Consistency models form a spectrum — the strongest guarantees (left) cost the most in coordination and availability; the weakest (right) maximize availability and latency.*

A client retrying a payment request with an idempotency key avoids a double charge.

```mermaid
sequenceDiagram
    participant Client
    participant Server
    participant Store as Idempotency Store (Redis/DB)
    participant Payments as Payment Processor

    Client->>Server: POST /payments (Idempotency-Key: abc123)
    Server->>Store: Lookup key "abc123"
    Store-->>Server: Not found
    Server->>Payments: Charge $50
    Payments-->>Server: Success (charge_id: ch_1)
    Server->>Store: Save {abc123 -> ch_1}
    Server-->>Client: 200 OK (charge_id: ch_1)

    Note over Client,Server: Network drops before client sees the response

    Client->>Server: POST /payments (Idempotency-Key: abc123) [retry]
    Server->>Store: Lookup key "abc123"
    Store-->>Server: Found -> ch_1
    Server-->>Client: 200 OK (charge_id: ch_1, cached)
    Note over Payments: Payment processor never called again — no double charge
```
*Caption: The server deduplicates retries by idempotency key, returning the original result instead of reprocessing the payment.*

Causal consistency: a reply must never be visible before the comment it responds to.

```mermaid
sequenceDiagram
    participant UserA as User A
    participant NodeX as Replica X
    participant NodeY as Replica Y
    participant UserB as User B

    UserA->>NodeX: Post comment "C1"
    NodeX-->>UserA: Ack
    UserA->>NodeX: Post reply "R1" (depends on C1)
    NodeX-->>UserA: Ack
    NodeX->>NodeY: Replicate C1, then R1 (causal order preserved)
    UserB->>NodeY: Read feed
    NodeY-->>UserB: [C1, R1] (never R1 without C1)
```
*Caption: Causal consistency guarantees that a "happens-before" relationship (reply depends on comment) is preserved across all replicas, even though unrelated events may still arrive out of order.*
