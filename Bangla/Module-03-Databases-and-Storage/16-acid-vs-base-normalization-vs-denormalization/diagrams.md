# Diagrams: ACID vs BASE, Normalization vs Denormalization

## ১. Normalized Relational Schema

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

*Caption: একটি normalized schema প্রতিটি তথ্য একবার সংরক্ষণ করে (user address, product price) এবং foreign key-এর মাধ্যমে রেকর্ড লিঙ্ক করে — আপডেট ঠিক একটি জায়গায় ঘটে।*

## ২. Denormalized Document Model (একই Data)

```mermaid
flowchart TD
    A["Order Document<br/>order_id: 1001<br/>status: shipped<br/>created_at: 2026-08-30"]
    A --> B["Embedded User Snapshot<br/>name: Alice<br/>email: alice@example.com<br/>address: 12 Elm St"]
    A --> C["Embedded Items Array"]
    C --> D["Item 1<br/>product_name: Wireless Mouse<br/>price: 19.99<br/>quantity: 2"]
    C --> E["Item 2<br/>product_name: USB-C Cable<br/>price: 9.99<br/>quantity: 1"]
```

*Caption: একটি denormalized document অর্ডারের ভেতরেই সরাসরি user এবং product data embed করে — একটি read-এই সবকিছু পাওয়া যায়, কিন্তু Alice-এর ঠিকানা এখন তার প্রতিটি অর্ডারে নকল হয়ে গেছে।*

## ৩. ACID Transaction Lifecycle vs Eventual Consistency Propagation

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

*Caption: ACID write স্বীকার করার আগেই correctness নিশ্চিত করে (reader-রা কখনো partial state দেখে না); BASE সাথে সাথেই স্বীকার করে নেয় এবং correctness-কে সময়ের সাথে replica জুড়ে ধরে ফেলতে দেয়।*
