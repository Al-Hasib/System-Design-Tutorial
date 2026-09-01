# Diagrams: Load Balancing

## 1. Basic Load Balancer Topology

```mermaid
flowchart TB
    Client1[Client A] --> LB[Load Balancer]
    Client2[Client B] --> LB
    Client3[Client C] --> LB
    LB --> S1[Server 1]
    LB --> S2[Server 2]
    LB --> S3[Server 3]
```
*Clients only ever address the load balancer; it decides which backend server handles each request, hiding the fleet's size and topology.*

## 2. Layer 4 vs Layer 7 Decision Making

```mermaid
flowchart LR
    subgraph L4["Layer 4 Load Balancer (Transport)"]
        direction LR
        R1["Sees: src/dst IP + port only"] --> D1["Forwards packets to a server"]
    end
    subgraph L7["Layer 7 Load Balancer (Application)"]
        direction LR
        R2["Sees: full HTTP request - path, headers, cookies"] --> D2{"Route by content"}
        D2 -->|"/api/images/*"| SvcA[Image Service Pool]
        D2 -->|"/api/checkout/*"| SvcB[Checkout Service Pool]
    end
```
*L4 balancers move packets based on IP/port alone; L7 balancers understand HTTP and can route different paths to entirely different backend pools.*

## 3. Health Check Loop Removing a Failed Server

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant S1 as Server 1 (healthy)
    participant S2 as Server 2 (failing)

    loop Every N seconds
        LB->>S1: GET /health
        S1-->>LB: 200 OK
        LB->>S2: GET /health
        S2-->>LB: Timeout / 500
    end
    Note over LB,S2: Server 2 marked unhealthy, removed from rotation
    LB->>S1: All new traffic routed here
```
*The load balancer continuously probes backend health and silently reroutes traffic away from failing servers.*
