# Diagrams: SQL vs NoSQL

## 1. Relational Schema — Normalized with Foreign Keys

```mermaid
erDiagram
    USERS ||--o{ ORDERS : places
    ORDERS ||--|{ ORDER_ITEMS : contains
    PRODUCTS ||--o{ ORDER_ITEMS : "referenced by"

    USERS {
        int id PK
        string name
        string email
    }
    ORDERS {
        int id PK
        int user_id FK
        date created_at
    }
    ORDER_ITEMS {
        int id PK
        int order_id FK
        int product_id FK
        int quantity
    }
    PRODUCTS {
        int id PK
        string title
        decimal price
    }
```
*Caption: A relational (SQL) schema stores each fact once and links tables with foreign keys, joined together at query time.*

## 2. Denormalized Document Structure (NoSQL)

```mermaid
flowchart TD
    Doc["Order Document (JSON)"]
    Doc --> A["order_id: 5031"]
    Doc --> B["user: { id: 12, name: 'Ana', email: '...' }"]
    Doc --> C["items: [ { product: 'Keyboard', price: 49.99, qty: 1 }, { product: 'Mouse', price: 19.99, qty: 2 } ]"]
    Doc --> D["created_at: 2026-08-30"]
```
*Caption: A document store embeds related data directly inside one record, trading duplication for a single fast read with no joins.*

## 3. Decision Tree: SQL vs NoSQL

```mermaid
flowchart TD
    Start["What are your data & access needs?"] --> Q1{"Is data highly relational\nwith complex, ad-hoc queries?"}
    Q1 -->|Yes| SQL["Use SQL\n(PostgreSQL, MySQL)"]
    Q1 -->|No| Q2{"Do you need massive\nhorizontal write scale?"}
    Q2 -->|Yes| Q3{"What's the data shape?"}
    Q2 -->|No| Q4{"Is it a simple key lookup\n(cache, session, cart)?"}
    Q3 -->|"Nested objects"| Doc["Document Store\n(MongoDB)"]
    Q3 -->|"High-volume events/time-series"| WideCol["Wide-Column Store\n(Cassandra)"]
    Q3 -->|"Relationship traversal"| Graph["Graph Database\n(Neo4j)"]
    Q4 -->|Yes| KV["Key-Value Store\n(Redis, DynamoDB)"]
    Q4 -->|No| SQL
```
*Caption: A practical decision path from data shape and scale requirements to a concrete database category.*
