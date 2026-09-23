# ডায়াগ্রাম: Distributed Caching with Redis & Memcached

## ১. Local Cache বনাম Distributed Cache স্থাপত্য

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

*Caption: Local cache প্রতিটি server-এ পৃথক, অসামঞ্জস্যপূর্ণ কপি তৈরি করে; একটি distributed cache প্রতিটি server-কে cached data-এর একই shared দৃশ্য দেয়।*

## ২. Distributed Cache Read/Write Flow (একটি Redis Cluster-এর সাথে Cache-Aside)

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

*Caption: Application server distributed cache cluster-কে একটি একক shared endpoint হিসেবে গণ্য করে, এর উপরে cache-aside logic প্রয়োগ করে।*

## ৩. একটি Cache Cluster জুড়ে Key Sharding করা (Consistent Hashing)

```mermaid
flowchart LR
    K1[Key: user:123] -->|hash| Ring((Hash Ring))
    K2[Key: user:456] -->|hash| Ring
    K3[Key: session:abc] -->|hash| Ring
    Ring --> N1[Cache Node A]
    Ring --> N2[Cache Node B]
    Ring --> N3[Cache Node C]
```

*Caption: Key-গুলো একটি ring-এ hash করা হয় এবং নিকটতম cache node-এ map করা হয়, তাই একটি node যোগ বা বাদ দিলে শুধু অল্প কিছু key-ই পুনরায় map হয়।*
