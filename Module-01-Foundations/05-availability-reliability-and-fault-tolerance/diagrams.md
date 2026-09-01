# Diagrams: Availability, Reliability, Redundancy & Fault Tolerance

## 1. Single Point of Failure vs Redundant Design

```mermaid
flowchart TD
    subgraph SPOF["No Redundancy (Single Point of Failure)"]
        C1[Client] --> S1[Single Server] --> D1[(Single Database)]
    end
```
*Caption: A single server and single database mean any one failure takes the whole system down.*

```mermaid
flowchart TD
    subgraph Redundant["Redundant Design"]
        C2[Client] --> LB[Load Balancer]
        LB --> S2[Server A]
        LB --> S3[Server B]
        S2 --> D2[(Primary DB)]
        S3 --> D2
        D2 -.replication.-> D3[(Replica DB)]
    end
```
*Caption: Redundant servers and a replicated database mean no single failure brings down the system.*

## 2. Failover Sequence via Health Checks

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant S1 as Server A (healthy)
    participant S2 as Server B (fails)
    LB->>S2: Health check
    S2--xLB: No response (down)
    LB->>S1: Route traffic here instead
    Note over LB,S2: Server B removed from rotation until healthy again
```
*Caption: Health checks let a load balancer detect a failed server and automatically fail traffic over to a healthy one.*

## 3. The Nines: Downtime Comparison

```mermaid
flowchart LR
    A["99%\n~3.65 days/year"] --> B["99.9%\n~8.76 hours/year"]
    B --> C["99.99%\n~52.6 minutes/year"]
    C --> D["99.999%\n~5.26 minutes/year"]
```
*Caption: Each additional "nine" of availability represents roughly a 10x reduction in allowed annual downtime.*
