# Diagrams: LSM Trees vs. B-Trees

## ১. B-Tree Write Path: Seek করে In Place Modify করা

```mermaid
flowchart LR
    W[Write request:<br/>update key 42] --> F[Traverse tree to find page containing key 42]
    F --> S["Seek to that page's location on disk"]
    S --> M[Modify page in place]
    M --> Split{Page now full?}
    Split -->|Yes| SP[Split page, rebalance tree]
    Split -->|No| Done[Write complete]
```

*প্রতিটি write-এর জন্য সেই key ধারণকারী নির্দিষ্ট page খুঁজে বের করতে হয়, তারপর সেটিকে in place modify করতে হয় — একটি random-access operation যা disk-এর যেকোনো জায়গায় ঘটতে পারে।*

## ২. LSM Tree Write Path: শুধু Append

```mermaid
flowchart LR
    W[Write request] --> WAL["Append to Write-Ahead Log<br/>(sequential I/O)"]
    WAL --> MT["Insert into in-memory Memtable<br/>(sorted structure)"]
    MT --> Full{Memtable full?}
    Full -->|Yes| Flush["Flush as new immutable SSTable<br/>(sequential write to disk)"]
    Full -->|No| Done[Write complete]
```

*প্রতিটি write একটি sequential append — write-ahead log এবং in-memory memtable উভয়েই — disk-এর কোনো নির্দিষ্ট location-এ seek করার প্রয়োজন হয় না।*

## ৩. LSM Tree Read Path এবং Compaction

```mermaid
flowchart TB
    R[Read request: key 42] --> MT2[Check Memtable]
    MT2 -->|Not found| S1["Check SSTable 3 (newest)<br/>Bloom filter says maybe present"]
    S1 -->|Not found| S2["Check SSTable 2<br/>Bloom filter says definitely absent, skip"]
    S2 -.->|skipped| S3["Check SSTable 1 (oldest)<br/>Found! Return this version"]

    subgraph Background["Background Compaction"]
        direction LR
        C1[SSTable 1] --> Merge[Merge + discard old/deleted versions]
        C2[SSTable 2] --> Merge
        Merge --> C3[New, larger SSTable]
    end
```

*একটি read প্রথমে memtable check করে, তারপর SSTable-গুলো newest থেকে oldest পর্যন্ত check করে, Bloom filter ব্যবহার করে সেসব SSTable বাদ দেয় যেগুলোতে প্রমাণিতভাবে সেই key নেই। Compaction background-এ চলে, SSTable-গুলোকে merge করে এবং stale/deleted data বাদ দিয়ে read amplification এবং space usage নিয়ন্ত্রণে রাখে।*
</content>
