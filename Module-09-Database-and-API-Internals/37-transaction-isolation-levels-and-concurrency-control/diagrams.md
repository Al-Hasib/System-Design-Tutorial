# Diagrams: Transaction Isolation Levels & Concurrency Control

## 1. A Dirty Read Anomaly

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
*Under Read Uncommitted, B can read a value A never actually committed — if A rolls back, B has already used data that never really existed.*

## 2. Pessimistic Locking vs. Optimistic Concurrency Control

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
*Pessimistic locking blocks contenders upfront; optimistic concurrency lets everyone proceed and only checks for a conflict right before committing, retrying if one is found.*

## 3. MVCC: Readers See a Consistent Snapshot, Writers Don't Block Them

```mermaid
flowchart LR
    W[Writer Transaction] -->|Creates new version| V2["Row version 2<br/>(new, uncommitted)"]
    V1["Row version 1<br/>(committed, snapshot at T0)"] -.->|Still visible to| R1[Reader Transaction<br/>started before writer committed]
    V2 -->|Becomes visible only after commit| R2[New Reader Transaction<br/>started after writer committed]
```
*A reader that started before the writer's commit keeps seeing the old, consistent version of the row — it never blocks waiting for the writer, and the writer never blocks waiting for it.*
