# Diagrams: LSM Trees vs. B-Trees

## 1. B-Tree Write Path: Seek and Modify In Place

```mermaid
flowchart LR
    W[Write request:<br/>update key 42] --> F[Traverse tree to find page containing key 42]
    F --> S["Seek to that page's location on disk"]
    S --> M[Modify page in place]
    M --> Split{Page now full?}
    Split -->|Yes| SP[Split page, rebalance tree]
    Split -->|No| Done[Write complete]
```
*Every write requires finding the specific page holding that key, then modifying it in place — a random-access operation that can occur anywhere on disk.*

## 2. LSM Tree Write Path: Append Only

```mermaid
flowchart LR
    W[Write request] --> WAL["Append to Write-Ahead Log<br/>(sequential I/O)"]
    WAL --> MT["Insert into in-memory Memtable<br/>(sorted structure)"]
    MT --> Full{Memtable full?}
    Full -->|Yes| Flush["Flush as new immutable SSTable<br/>(sequential write to disk)"]
    Full -->|No| Done[Write complete]
```
*Every write is a sequential append — to the write-ahead log and the in-memory memtable — with no seeking to a specific disk location required.*

## 3. LSM Tree Read Path and Compaction

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
*A read checks the memtable, then SSTables from newest to oldest, using Bloom filters to skip SSTables that provably don't contain the key. Compaction runs in the background, merging SSTables and discarding stale/deleted data to keep read amplification and space usage in check.*
