# Diagrams: Database Replication

## 1. Master-Slave (Leader-Follower) Replication

```mermaid
flowchart TD
    Client1[Client: Write] -->|INSERT/UPDATE/DELETE| Master[(Master / Leader)]
    Client2[Client: Read] --> LB[Load Balancer]
    LB --> Follower1[(Follower / Replica 1)]
    LB --> Follower2[(Follower / Replica 2)]
    LB --> Master
    Master -->|Async / Sync replication| Follower1
    Master -->|Async / Sync replication| Follower2
```
*Caption: All writes go to a single master, which streams changes to followers; reads can be distributed across the master and any follower to scale read capacity.*

## 2. Master-Master (Multi-Leader) Replication

```mermaid
flowchart LR
    ClientUS[US Client] -->|Write| MasterUS[(Master US)]
    ClientEU[EU Client] -->|Write| MasterEU[(Master EU)]
    MasterUS <-->|Bidirectional replication| MasterEU
    MasterUS -->|Conflict?| Resolve[Conflict Resolution\nLWW / Vector Clocks]
    MasterEU -->|Conflict?| Resolve
```
*Caption: Multiple masters each accept writes locally and replicate to each other; concurrent writes to the same record must go through conflict resolution.*

## 3. Failover Sequence in Master-Slave Replication

```mermaid
sequenceDiagram
    participant App as Application
    participant M as Master
    participant F1 as Follower 1
    participant F2 as Follower 2

    App->>M: Write request
    M-->>F1: Replicate change
    M-->>F2: Replicate change
    Note over M: Master crashes
    App--xM: Write request fails
    Note over F1,F2: Failover detection & election
    F1->>F1: Promoted to new Master
    App->>F1: Write request (new master)
    F1-->>F2: Replicate change
```
*Caption: When the master fails, a follower must be detected as missing, elected, and promoted before writes can resume — this gap is the failover window.*
