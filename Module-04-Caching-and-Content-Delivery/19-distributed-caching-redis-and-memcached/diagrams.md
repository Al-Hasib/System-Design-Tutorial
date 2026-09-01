# Diagrams: Distributed Caching with Redis & Memcached

## 1. Local Cache vs Distributed Cache Architecture

```mermaid
flowchart TB
    subgraph Local Caches - Inconsistent
        A1[App Server 1 + Local Cache] 
        A2[App Server 2 + Local Cache]
        A3[App Server 3 + Local Cache]
    end

    subgraph Distributed Cache - Shared and Consistent
        B1[App Server 1] --> C[(Shared Redis/Memcached Cluster)]
        B2[App Server 2] --> C
        B3[App Server 3] --> C
    end
```

*Caption: Local caches create separate, inconsistent copies per server; a distributed cache gives every server the same shared view of cached data.*

## 2. Distributed Cache Read/Write Flow (Cache-Aside with a Redis Cluster)

```mermaid
sequenceDiagram
    participant App as Application Server
    participant Cache as Redis/Memcached Cluster
    participant DB as Database

    App->>Cache: GET user:123
    alt Cache Hit
        Cache-->>App: Return cached value
    else Cache Miss
        Cache-->>App: nil
        App->>DB: Query user 123
        DB-->>App: Return row
        App->>Cache: SET user:123 (with TTL)
        Cache-->>App: OK
    end
```

*Caption: Application servers treat the distributed cache cluster as a single shared endpoint, applying cache-aside logic on top of it.*

## 3. Sharding Keys Across a Cache Cluster (Consistent Hashing)

```mermaid
flowchart LR
    K1[Key: user:123] -->|hash| Ring((Hash Ring))
    K2[Key: user:456] -->|hash| Ring
    K3[Key: session:abc] -->|hash| Ring
    Ring --> N1[Cache Node A]
    Ring --> N2[Cache Node B]
    Ring --> N3[Cache Node C]
```

*Caption: Keys are hashed onto a ring and mapped to the nearest cache node, so adding or removing a node only remaps a small fraction of keys.*
