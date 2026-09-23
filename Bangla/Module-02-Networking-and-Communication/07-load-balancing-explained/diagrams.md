# Diagrams: Load Balancing

## ১. মৌলিক Load Balancer Topology

```mermaid
flowchart TB
    Client1[Client A] --> LB[Load Balancer]
    Client2[Client B] --> LB
    Client3[Client C] --> LB
    LB --> S1[Server 1]
    LB --> S2[Server 2]
    LB --> S3[Server 3]
```
*Client-রা কখনোই শুধুমাত্র load balancer-কে address করে; এটাই সিদ্ধান্ত নেয় কোন backend server প্রতিটা request handle করবে, fleet-এর আকার এবং topology লুকিয়ে রাখে।*

## ২. Layer 4 vs Layer 7 সিদ্ধান্ত গ্রহণ

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
*L4 balancer শুধু IP/port-এর ভিত্তিতে packet সরায়; L7 balancer HTTP বোঝে এবং ভিন্ন path-কে সম্পূর্ণ ভিন্ন backend pool-এ route করতে পারে।*

## ৩. একটা ব্যর্থ Server অপসারণকারী Health Check Loop

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
*Load balancer ক্রমাগত backend health probe করে এবং নিঃশব্দে ব্যর্থ server গুলো থেকে traffic দূরে reroute করে।*
