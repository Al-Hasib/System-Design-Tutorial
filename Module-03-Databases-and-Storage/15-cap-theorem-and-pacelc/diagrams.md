# Diagrams: CAP Theorem & PACELC

Mermaid has no native triangle/Venn shape, so the CAP relationship below is approximated as a graph of the three properties with the "pick two" pairings called out.

## 1. CAP Theorem — Pick Two of Three

```mermaid
graph TD
    C["Consistency (C)"]
    A["Availability (A)"]
    P["Partition Tolerance (P)"]

    C ---|"CP: consistent, may reject requests during a partition"| P
    A ---|"AP: available, may return stale data during a partition"| P
    C -.->|"CA only possible without a network partition — not realistic for multi-node systems"| A

    style C fill:#4C6EF5,color:#fff
    style A fill:#12B886,color:#fff
    style P fill:#F59F00,color:#fff
```

*Caption: A distributed system can only fully guarantee two of Consistency, Availability, and Partition Tolerance at once — since partitions are unavoidable, real systems must choose CP or AP.*

## 2. Read/Write During a Network Partition — CP vs. AP

```mermaid
sequenceDiagram
    participant Client
    participant NodeA as Node A (reachable)
    participant NodeB as Node B (unreachable, other side of partition)

    Note over NodeA,NodeB: Network partition occurs between Node A and Node B

    rect rgb(240, 230, 200)
    Note over Client,NodeB: CP System Behavior
    Client->>NodeA: Write/Read request
    NodeA->>NodeB: Attempt to confirm quorum/majority
    NodeB-->>NodeA: No response (partitioned)
    NodeA-->>Client: Error / request rejected (cannot confirm consistency)
    end

    rect rgb(210, 235, 225)
    Note over Client,NodeB: AP System Behavior
    Client->>NodeA: Write/Read request
    NodeA-->>Client: Response served locally (may be stale vs. Node B)
    Note over NodeA,NodeB: Reconciliation happens later once partition heals
    end
```

*Caption: In a CP system, Node A refuses the request when it can't confirm agreement with the partitioned Node B; in an AP system, Node A answers immediately and reconciles any divergence after the partition heals.*

## 3. PACELC Decision Flow

```mermaid
flowchart TD
    Start["Is the system currently experiencing a network Partition?"]
    Start -->|Yes| PChoice["Choose: Availability (A) or Consistency (C)"]
    Start -->|No, Else: normal operation| EChoice["Choose: Latency (L) or Consistency (C)"]

    PChoice --> PA["PA: keep serving requests\n(e.g., Cassandra, DynamoDB)"]
    PChoice --> PC["PC: reject/block requests\n(e.g., MongoDB majority, ZooKeeper)"]

    EChoice --> EL["EL: respond fast, weaker consistency\n(e.g., Cassandra, DynamoDB default)"]
    EChoice --> EC["EC: wait for agreement, stronger consistency\n(e.g., MongoDB majority, ZooKeeper)"]
```

*Caption: PACELC adds a second decision that applies even without a partition — trading Latency against Consistency during normal operation.*
