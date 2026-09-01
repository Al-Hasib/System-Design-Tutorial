# Diagrams: Caching Strategies & Cache Invalidation

## 1. Cache-Aside Read Flow

```mermaid
flowchart TD
    A[Application receives read request] --> B{Data in cache?}
    B -- Cache Hit --> C[Return data from cache]
    B -- Cache Miss --> D[Query database]
    D --> E[Write result into cache]
    E --> F[Return data to caller]
```

*Caption: In cache-aside, the application checks the cache first and only falls back to the database on a miss, populating the cache afterward.*

## 2. Write-Through vs Write-Back Sequence

```mermaid
sequenceDiagram
    participant App as Application
    participant Cache
    participant DB as Database

    Note over App,DB: Write-Through
    App->>Cache: Write data
    Cache->>DB: Write data (synchronous)
    DB-->>Cache: Ack
    Cache-->>App: Ack (write confirmed in both)

    Note over App,DB: Write-Back (Write-Behind)
    App->>Cache: Write data
    Cache-->>App: Ack (immediate)
    Cache->>DB: Flush write (asynchronous, later/batched)
    DB-->>Cache: Ack
```

*Caption: Write-through confirms only after both cache and DB are updated; write-back confirms immediately and flushes to the database asynchronously.*

## 3. Cache Stampede Mitigation with Request Coalescing

```mermaid
flowchart TD
    A[Hot key expires] --> B[Thousands of requests arrive simultaneously]
    B --> C{First request acquires lock?}
    C -- Yes --> D[Request fetches from DB and repopulates cache]
    C -- No, lock held --> E[Other requests wait briefly or receive stale/placeholder data]
    D --> F[Lock released, cache repopulated]
    F --> G[Subsequent requests served from fresh cache]
    E --> G
```

*Caption: Request coalescing ensures only one request repopulates a hot cache key while others wait or receive a stale fallback, protecting the database from a stampede.*
