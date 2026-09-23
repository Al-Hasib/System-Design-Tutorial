# Diagrams: Data Consistency Models ও Idempotency

Consistency spectrum, সবচেয়ে শক্তিশালী থেকে সবচেয়ে দুর্বল guarantee পর্যন্ত।

```mermaid
flowchart LR
    A["Linearizable\n(Strong)\nSingle total order,\nhigh coordination cost"] --> B["Causal Consistency\nHappens-before order\npreserved"]
    B --> C["Session Guarantees\nRead-your-writes,\nMonotonic reads/writes"]
    C --> D["Eventual Consistency\nConverges eventually,\nno ordering guarantee"]

    style A fill:#f96,stroke:#333
    style D fill:#9cf,stroke:#333
```

*Caption: Consistency model-গুলো একটি spectrum গঠন করে — সবচেয়ে শক্তিশালী guarantee (বামে) coordination এবং availability-তে সবচেয়ে বেশি খরচ করে; সবচেয়ে দুর্বলগুলো (ডানে) availability এবং latency সর্বোচ্চ করে।*

একটি idempotency key দিয়ে একটি payment request retry করা client double charge এড়ায়।

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

*Caption: Server idempotency key দিয়ে retry deduplicate করে, payment পুনরায় process না করে original result ফেরত দেয়।*

Causal consistency: একটি reply যে comment-এর প্রতিক্রিয়া, তার আগে কখনো দৃশ্যমান হওয়া উচিত নয়।

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

*Caption: Causal consistency guarantee করে যে একটি "happens-before" সম্পর্ক (reply, comment-এর উপর নির্ভরশীল) সব replica জুড়ে সংরক্ষিত থাকে, যদিও unrelated event তখনও out of order আসতে পারে।*
