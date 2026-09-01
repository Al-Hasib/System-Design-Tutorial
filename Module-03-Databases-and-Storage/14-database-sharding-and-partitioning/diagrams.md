# Diagrams: Database Sharding & Partitioning

## 1. Sharded Database Layout (Application → Router → Shards)

```mermaid
flowchart LR
    App[Application] --> Router[Shard Router / Directory Service]
    Router --> Shard1[(Shard 1<br/>user_id range/hash A)]
    Router --> Shard2[(Shard 2<br/>user_id range/hash B)]
    Router --> Shard3[(Shard 3<br/>user_id range/hash C)]
```

*The application never talks to shards directly — a router (or lookup/directory service) inspects the shard key on each request and forwards it to the one shard that owns that data.*

## 2. Range-Based vs Hash-Based Key Distribution

```mermaid
flowchart TB
    subgraph Range["Range-Based Sharding"]
        direction LR
        R0["IDs 1-1,000,000"] --> RS1[(Shard 1)]
        R1["IDs 1,000,001-2,000,000"] --> RS2[(Shard 2)]
        R2["IDs 2,000,001-3,000,000<br/>(newest, hottest)"] --> RS3[(Shard 3)]
    end

    subgraph Hash["Hash-Based Sharding"]
        direction LR
        H0["hash(id) % 3 == 0"] --> HS1[(Shard 1)]
        H1["hash(id) % 3 == 1"] --> HS2[(Shard 2)]
        H2["hash(id) % 3 == 2"] --> HS3[(Shard 3)]
    end
```

*Range-based sharding keeps contiguous keys together (great for range scans, but new/sequential traffic piles onto one shard); hash-based sharding scatters keys pseudo-randomly (even load, but no cheap range scans).*

## 3. Directory-Based Sharding Lookup

```mermaid
flowchart LR
    App2[Application] -->|1 . lookup key| Dir[Directory Service<br/>key to shard mapping]
    Dir -->|2 . returns shard 2| App2
    App2 -->|3 . query| Shard2b[(Shard 2)]
```

*Directory-based sharding adds one extra network hop to resolve "which shard owns this key," but in exchange lets individual keys be moved or rebalanced freely without a rigid formula.*
