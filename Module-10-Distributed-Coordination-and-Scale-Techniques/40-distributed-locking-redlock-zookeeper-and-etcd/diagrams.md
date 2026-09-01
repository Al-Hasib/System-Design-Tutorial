# Diagrams: Distributed Locking

## 1. Redlock: Majority Across Independent Redis Instances

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
*Redlock only needs a majority of independent instances to agree, so it tolerates some instances being slow or unreachable — the trade-off is that its safety depends on timing assumptions across those instances.*

## 2. ZooKeeper-Style Lock via Sequential Ephemeral Nodes

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
*Each contender creates a sequential node; whoever has the lowest number holds the lock, and the next-in-line watches for it to disappear — either on release or automatically if that client's session dies.*

## 3. The Fencing Token: Preventing a Stale Holder from Causing Damage

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
*Even though Client A's lock check originally succeeded, its stale fencing token is rejected at the point of actually writing — the resource itself, not just the lock, enforces correctness.*
