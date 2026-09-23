# Diagrams: Distributed Transactions — 2PC ও Saga

## ১. Two-Phase Commit (2PC)

```mermaid
sequenceDiagram
    participant C as Coordinator
    participant P1 as Participant: Order DB
    participant P2 as Participant: Payment DB
    participant P3 as Participant: Inventory DB

    Note over C,P3: Phase 1 - Prepare / Vote
    C->>P1: PREPARE
    C->>P2: PREPARE
    C->>P3: PREPARE
    P1-->>C: VOTE YES (locked, logged)
    P2-->>C: VOTE YES (locked, logged)
    P3-->>C: VOTE YES (locked, logged)

    Note over C,P3: Phase 2 - Commit (all voted yes)
    C->>P1: COMMIT
    C->>P2: COMMIT
    C->>P3: COMMIT
    P1-->>C: ACK
    P2-->>C: ACK
    P3-->>C: ACK

    Note over C,P3: If Coordinator crashes here after PREPARE,<br/>all participants stay blocked holding locks
```

*Caption: Coordinator commit করার আগে 2PC প্রতিটি participant থেকে একটি সর্বসম্মত "হ্যাঁ" vote দাবি করে; Prepare এবং Commit-এর মধ্যে একটি coordinator crash participant-দের block অবস্থায় রেখে দেয়।*

## ২. Orchestration-Based Saga (Failure-এ Compensation-সহ)

```mermaid
sequenceDiagram
    participant O as Order Service
    participant Orch as Saga Orchestrator
    participant Pay as Payment Service
    participant Inv as Inventory Service
    participant Ship as Shipping Service

    O->>Orch: Start Checkout Saga
    Orch->>Pay: Charge Payment
    Pay-->>Orch: Payment Committed
    Orch->>Inv: Reserve Inventory
    Inv-->>Orch: Inventory FAILED (out of stock)

    Note over Orch: Failure detected - begin compensation (reverse order)
    Orch->>Pay: Compensate: Refund Payment
    Pay-->>Orch: Refund Committed
    Orch->>O: Compensate: Cancel Order
    O-->>Orch: Order Cancelled

    Note over Orch,Ship: Shipping Service never invoked - saga stopped before reaching it
```

*Caption: একটি central orchestrator ক্রমানুসারে প্রতিটি local transaction চালায় এবং downstream-এর কোনো step ব্যর্থ হলে explicitly compensating transaction, বিপরীত ক্রমে, ট্রিগার করে।*

## ৩. Choreography-Based Saga (Event-Driven, কোনো Central Orchestrator নেই)

```mermaid
sequenceDiagram
    participant O as Order Service
    participant Bus as Event Bus
    participant Pay as Payment Service
    participant Inv as Inventory Service

    O->>Bus: publish OrderCreated
    Bus->>Pay: OrderCreated
    Pay->>Pay: Charge Card (local tx)
    Pay->>Bus: publish PaymentCompleted
    Bus->>Inv: PaymentCompleted
    Inv->>Inv: Reserve Stock (local tx) - FAILS
    Inv->>Bus: publish InventoryReservationFailed

    Bus->>Pay: InventoryReservationFailed
    Pay->>Pay: Refund Card (compensating tx)
    Pay->>Bus: publish PaymentRefunded
    Bus->>O: PaymentRefunded
    O->>O: Mark Order Cancelled (compensating tx)
```

*Caption: Choreography-তে, প্রতিটি service স্বাধীনভাবে event-এর প্রতিক্রিয়া জানায় এবং failure-এর সময় নিজের compensating transaction চালায় — সামগ্রিক saga state ট্র্যাক করার কোনো central coordinator নেই।*
