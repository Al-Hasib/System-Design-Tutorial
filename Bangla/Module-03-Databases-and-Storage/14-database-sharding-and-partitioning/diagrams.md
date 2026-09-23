# Diagrams: Database Sharding & Partitioning

## ১. Sharded Database Layout (Application → Router → Shards)

```mermaid
flowchart LR
    App[Application] --> Router[Shard Router / Directory Service]
    Router --> Shard1[(Shard 1<br/>user_id range/hash A)]
    Router --> Shard2[(Shard 2<br/>user_id range/hash B)]
    Router --> Shard3[(Shard 3<br/>user_id range/hash C)]
```

*Application কখনও সরাসরি shard-এর সাথে কথা বলে না — একটি router (বা lookup/directory service) প্রতিটি request-এ shard key পরীক্ষা করে এবং যে একটি shard সেই ডেটার মালিক তার কাছে forward করে।*

## ২. Range-Based বনাম Hash-Based Key Distribution

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

*Range-based sharding contiguous key-গুলোকে একসাথে রাখে (range scan-এর জন্য চমৎকার, কিন্তু নতুন/sequential traffic একটি shard-এ জমা হয়ে যায়); hash-based sharding key-গুলোকে pseudo-randomly ছড়িয়ে দেয় (সমান load, কিন্তু সস্তা range scan নেই)।*

## ৩. Directory-Based Sharding Lookup

```mermaid
flowchart LR
    App2[Application] -->|1 . lookup key| Dir[Directory Service<br/>key to shard mapping]
    Dir -->|2 . returns shard 2| App2
    App2 -->|3 . query| Shard2b[(Shard 2)]
```

*Directory-based sharding "কোন shard এই key-এর মালিক" তা resolve করতে একটি অতিরিক্ত network hop যোগ করে, কিন্তু এর বিনিময়ে individual key-গুলোকে কোনো কঠোর formula ছাড়াই স্বাধীনভাবে সরানো বা rebalance করতে দেয়।*
