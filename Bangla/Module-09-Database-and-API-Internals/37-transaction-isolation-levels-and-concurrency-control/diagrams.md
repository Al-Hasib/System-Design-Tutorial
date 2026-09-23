# Diagrams: Transaction Isolation Levels & Concurrency Control

## ১. একটি Dirty Read Anomaly

```mermaid
sequenceDiagram
    participant A as Transaction A
    participant DB as Database Row (balance = 100)
    participant B as Transaction B

    A->>DB: UPDATE balance = 150 (not yet committed)
    B->>DB: READ balance
    DB-->>B: 150 (dirty read!)
    A->>DB: ROLLBACK
    Note over B: B acted on a value that never actually existed
```

*Read Uncommitted-এর অধীনে, B এমন একটি value read করতে পারে যা A আসলে কখনো commit করেনি — যদি A rollback করে, তাহলে B ইতিমধ্যে এমন ডেটা ব্যবহার করে ফেলেছে যা আসলে কখনো ছিলই না।*

## ২. Pessimistic Locking vs. Optimistic Concurrency Control

```mermaid
flowchart TB
    subgraph Pessimistic["Pessimistic Locking"]
        P1[Transaction starts] --> P2[Acquire lock on row]
        P2 --> P3[Other transactions wait]
        P3 --> P4[Modify row]
        P4 --> P5[Commit, release lock]
    end

    subgraph Optimistic["Optimistic Concurrency Control"]
        O1[Transaction starts] --> O2[Read row + version number]
        O2 --> O3[No lock — other transactions proceed freely]
        O3 --> O4{Version still matches at commit?}
        O4 -->|Yes| O5[Commit succeeds]
        O4 -->|No, someone else changed it| O6[Abort — application retries]
    end
```

*Pessimistic locking প্রতিদ্বন্দ্বীদের আগে থেকেই block করে দেয়; optimistic concurrency সবাইকে এগিয়ে যেতে দেয় এবং শুধু commit করার ঠিক আগে conflict চেক করে, একটি পাওয়া গেলে retry করে।*

## ৩. MVCC: Reader-রা একটি Consistent Snapshot দেখে, Writer-রা তাদের Block করে না

```mermaid
flowchart LR
    W[Writer Transaction] -->|Creates new version| V2["Row version 2<br/>(new, uncommitted)"]
    V1["Row version 1<br/>(committed, snapshot at T0)"] -.->|Still visible to| R1[Reader Transaction<br/>started before writer committed]
    V2 -->|Becomes visible only after commit| R2[New Reader Transaction<br/>started after writer committed]
```

*writer commit করার আগে শুরু হওয়া একটি reader row-টির পুরোনো, consistent version দেখতে থাকে — এটি কখনো writer-এর জন্য অপেক্ষা করে block হয় না, এবং writer-ও কখনো এর জন্য অপেক্ষা করে block হয় না।*
