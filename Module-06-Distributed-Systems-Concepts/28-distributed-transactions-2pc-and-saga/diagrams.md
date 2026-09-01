# Diagrams: Distributed Transactions — 2PC & Saga

## 1. Two-Phase Commit (2PC)

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

*Caption: 2PC requires a unanimous "yes" vote from every participant before the coordinator commits; a coordinator crash between Prepare and Commit leaves participants blocked.*

## 2. Orchestration-Based Saga (with Compensation on Failure)

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

*Caption: A central orchestrator drives each local transaction in sequence and explicitly triggers compensating transactions, in reverse order, when a downstream step fails.*

## 3. Choreography-Based Saga (Event-Driven, No Central Orchestrator)

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

*Caption: In choreography, each service reacts to events independently and runs its own compensating transaction on failure — there is no central coordinator tracking the overall saga state.*
