# Diagrams: Logical Clocks & Time in Distributed Systems

## 1. Lamport Timestamps: Happened-Before Ordering

```mermaid
sequenceDiagram
    participant P1 as Process 1
    participant P2 as Process 2

    Note over P1: Local event, counter = 1
    P1->>P2: Message (carries counter = 1)
    Note over P2: Local event, counter = 1
    Note over P2: Receives message, counter = max(1,1)+1 = 2
    Note over P2: Local event, counter = 3
    P2->>P1: Message (carries counter = 3)
    Note over P1: Receives message, counter = max(1,3)+1 = 4
```
*Every local event increments a process's own counter; receiving a message bumps the counter above both the local and received values — guaranteeing any causally-earlier event gets a smaller number.*

## 2. Vector Clocks: Detecting True Concurrency

```mermaid
flowchart TB
    subgraph Causal["Case 1: Causally Related"]
        VA["Vector A = [2,0]"] -.->|"every element of A <= B"| VB["Vector B = [3,1]"]
        R1["A happened-before B"]
    end

    subgraph Concurrent["Case 2: Truly Concurrent"]
        VC["Vector C = [3,0]"]
        VD["Vector D = [0,2]"]
        R2["Neither dominates the other - C and D are concurrent (a real conflict)"]
    end
```
*If one vector dominates the other in every slot, the events are causally ordered. If neither dominates, the events happened independently — a genuine conflict, not something a single "which came first" answer can resolve.*

## 3. Vector Clocks Resolving a Shopping Cart Conflict

```mermaid
sequenceDiagram
    participant Phone as Phone (writes to US DC)
    participant US as US Data Center
    participant EU as EU Data Center
    participant Laptop as Laptop (writes to EU DC, stale view)

    Phone->>US: Add item X, vector [1,0]
    Laptop->>EU: Add item Y, vector [0,1] (unaware of Phone's update)
    US->>EU: Replicate vector [1,0]
    EU->>EU: Compare [1,0] vs [0,1] - neither dominates - CONCURRENT
    EU->>EU: Merge both updates instead of picking one via timestamp
```
*Because neither write's vector clock dominates the other, the system correctly identifies this as a genuine conflict and merges both changes rather than silently discarding one based on an untrustworthy wall-clock timestamp.*
