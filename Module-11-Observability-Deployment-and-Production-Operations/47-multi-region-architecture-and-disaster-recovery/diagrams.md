# Diagrams: Multi-Region Architecture & Disaster Recovery

## 1. Active-Passive Failover

```mermaid
flowchart TB
    subgraph Before["Before Disaster"]
        U1[Users] --> A1["Region A (Active)<br/>Serving all traffic"]
        A1 -.->|Replicate data| P1["Region B (Passive)<br/>Standby, not serving traffic"]
    end

    subgraph After["After Region A Fails"]
        U2[Users] --> F["Failover:<br/>DNS/routing updated"]
        F --> P2["Region B (now Active)<br/>Serving all traffic"]
    end
```
*Under normal operation, Region B sits idle with replicated data. After a failure, traffic is redirected to it — but that redirection itself takes time, which RTO defines an acceptable limit for.*

## 2. Active-Active: Every Region Serves Traffic

```mermaid
flowchart TB
    U1[Users near Region A] --> A["Region A<br/>Serving live traffic"]
    U2[Users near Region B] --> B["Region B<br/>Serving live traffic"]
    A <-->|Bidirectional replication| B

    Note["If Region A fails,<br/>Region B simply absorbs its traffic<br/>- no failover delay"]
```
*Both regions serve traffic simultaneously, routed by proximity — if one fails, there's no single point that needs to "take over," but writes accepted in both places must be reconciled.*

## 3. RTO and RPO on a Timeline

```mermaid
flowchart LR
    LastBackup["Last successful<br/>replicated write"] -->|"RPO window<br/>(max acceptable data loss)"| Disaster["Disaster occurs"]
    Disaster -->|"RTO window<br/>(max acceptable downtime)"| Recovered["System back online,<br/>serving traffic again"]
```
*RPO measures backward from the disaster — how much recent data could be lost. RTO measures forward from the disaster — how long until service is restored. Both are business-defined targets that drive the actual architecture choice.*
