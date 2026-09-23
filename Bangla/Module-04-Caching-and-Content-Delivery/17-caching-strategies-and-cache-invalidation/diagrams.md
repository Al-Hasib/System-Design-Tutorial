# ডায়াগ্রাম: Caching Strategies & Cache Invalidation

## ১. Cache-Aside Read Flow

```mermaid
flowchart TD
    A[Application receives read request] --> B{Data in cache?}
    B -- Cache Hit --> C[Return data from cache]
    B -- Cache Miss --> D[Query database]
    D --> E[Write result into cache]
    E --> F[Return data to caller]
```

*Caption: cache-aside-এ, অ্যাপ্লিকেশন প্রথমে cache চেক করে এবং শুধুমাত্র miss হলে database-এর দিকে ফিরে যায়, পরে cache পূরণ করে।*

## ২. Write-Through বনাম Write-Back Sequence

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

*Caption: write-through শুধুমাত্র cache এবং DB উভয়ই আপডেট হওয়ার পরে নিশ্চিত করে; write-back তাৎক্ষণিকভাবে নিশ্চিত করে এবং asynchronously database-এ flush করে।*

## ৩. Request Coalescing দিয়ে Cache Stampede প্রশমন

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

*Caption: request coalescing নিশ্চিত করে যে শুধুমাত্র একটি request একটি hot cache key পুনরায় পূরণ করে, যখন বাকিগুলো অপেক্ষা করে বা একটি stale fallback পায়, এভাবে database-কে একটি stampede থেকে রক্ষা করে।*
