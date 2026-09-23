# Diagrams: Distributed Locking

## ১. Redlock: Independent Redis Instance জুড়ে Majority

```mermaid
flowchart TB
    C[Client requests lock] --> R1[Redis Instance 1: acquired]
    C --> R2[Redis Instance 2: acquired]
    C --> R3[Redis Instance 3: acquired]
    C --> R4["Redis Instance 4: timeout/unreachable"]
    C --> R5["Redis Instance 5: timeout/unreachable"]
    R1 --> M{"Majority (3 of 5) acquired within time budget?"}
    R2 --> M
    R3 --> M
    M -->|Yes| Granted[Lock considered granted]
    M -->|No| Failed[Lock acquisition fails]
```
*Redlock-এর কেবল independent instance-গুলোর একটি majority-র একমত হওয়া প্রয়োজন, তাই এটি কিছু instance ধীর বা unreachable হলেও সহ্য করতে পারে — বিনিময়ে এর safety সেই instance-গুলো জুড়ে timing assumption-এর ওপর নির্ভর করে।*

## ২. Sequential Ephemeral Node-এর মাধ্যমে ZooKeeper-Style Lock

```mermaid
sequenceDiagram
    participant A as Client A
    participant B as Client B
    participant ZK as ZooKeeper Cluster

    A->>ZK: Create sequential ephemeral node (lock-0001)
    B->>ZK: Create sequential ephemeral node (lock-0002)
    Note over A,ZK: A has lowest sequence number - A holds the lock
    B->>ZK: Watch node lock-0001
    A->>A: Do protected work
    A->>ZK: Release (delete lock-0001)
    ZK-->>B: Notify: lock-0001 removed
    Note over B,ZK: B now has lowest sequence number - B holds the lock
```
*প্রতিটি প্রতিদ্বন্দ্বী একটি sequential node তৈরি করে; যার সংখ্যা সবচেয়ে কম সে lock ধরে রাখে, এবং সারিতে পরবর্তী জন এটি অদৃশ্য হওয়ার জন্য watch করে — হয় release-এ, অথবা সেই client-এর session মারা গেলে স্বয়ংক্রিয়ভাবে।*

## ৩. Fencing Token: একজন Stale Holder-কে ক্ষতি করা থেকে প্রতিরোধ করা

```mermaid
sequenceDiagram
    participant A as Client A
    participant Lock as Lock Service
    participant Storage as Shared Storage

    A->>Lock: Acquire lock
    Lock-->>A: Granted, fencing token = 33
    Note over A: Long pause (GC, network delay)
    Note over Lock: Lock expires
    participant B as Client B
    B->>Lock: Acquire lock
    Lock-->>B: Granted, fencing token = 34
    B->>Storage: Write (token 34)
    Storage-->>B: Accepted, latest token now 34
    A->>Storage: Write (token 33, resumed after pause)
    Storage-->>A: Rejected - token 33 is older than 34
```
*যদিও Client A-এর lock check মূলত সফল হয়েছিল, এর stale fencing token প্রকৃতপক্ষে লেখার মুহূর্তে প্রত্যাখ্যাত হয় — resource নিজেই, শুধু lock নয়, correctness প্রয়োগ করে।*
