# Diagrams: ACID vs BASE, Normalization vs Denormalization

## 1. Normalized Relational Schema

```mermaid
erDiagram
    USERS ||--o{ ORDERS : places
    ORDERS ||--|{ ORDER_ITEMS : contains
    PRODUCTS ||--o{ ORDER_ITEMS : "referenced by"

    USERS {
        int user_id PK
        string name
        string email
        string address
    }
    ORDERS {
        int order_id PK
        int user_id FK
        datetime created_at
        string status
    }
    ORDER_ITEMS {
        int order_item_id PK
        int order_id FK
        int product_id FK
        int quantity
    }
    PRODUCTS {
        int product_id PK
        string name
        decimal price
    }
```

*Caption: A normalized schema stores each fact once (user address, product price) and links records via foreign keys — updates happen in exactly one place.*

## 2. Denormalized Document Model (Same Data)

```mermaid
flowchart TD
    A["Order Document<br/>order_id: 1001<br/>status: shipped<br/>created_at: 2026-08-30"]
    A --> B["Embedded User Snapshot<br/>name: Alice<br/>email: alice@example.com<br/>address: 12 Elm St"]
    A --> C["Embedded Items Array"]
    C --> D["Item 1<br/>product_name: Wireless Mouse<br/>price: 19.99<br/>quantity: 2"]
    C --> E["Item 2<br/>product_name: USB-C Cable<br/>price: 9.99<br/>quantity: 1"]
```

*Caption: A denormalized document embeds user and product data directly inside the order — one read returns everything, but Alice's address is now duplicated across every order she has ever placed.*

## 3. ACID Transaction Lifecycle vs Eventual Consistency Propagation

```mermaid
flowchart LR
    subgraph ACID["ACID Transaction (single strongly-consistent store)"]
        direction LR
        A1["BEGIN transaction"] --> A2["Debit Alice -$100"]
        A2 --> A3["Credit Bob +$100"]
        A3 --> A4{"All checks pass?"}
        A4 -- "Yes" --> A5["COMMIT (durable, visible to all readers immediately)"]
        A4 -- "No" --> A6["ROLLBACK (nothing changed)"]
    end

    subgraph BASE["BASE / Eventual Consistency (replicated store)"]
        direction LR
        B1["Write lands on Node 1"] --> B2["Node 1 acknowledges write<br/>(basically available)"]
        B2 --> B3["Async replication begins"]
        B3 --> B4["Node 2 still has stale value<br/>(soft state)"]
        B3 --> B5["Node 3 still has stale value<br/>(soft state)"]
        B4 --> B6["Replication completes"]
        B5 --> B6
        B6 --> B7["All nodes converge<br/>(eventual consistency)"]
    end
```

*Caption: ACID resolves correctness before acknowledging the write (readers never see a partial state); BASE acknowledges immediately and lets correctness catch up across replicas over time.*
