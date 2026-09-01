# Diagrams: Scalability Basics — Vertical vs Horizontal Scaling

## 1. Vertical Scaling

```mermaid
flowchart LR
    A[Server\n2 CPU / 4GB RAM] -->|Upgrade| B[Same Server\n16 CPU / 64GB RAM]
```
*Caption: Vertical scaling replaces a machine with a more powerful version of itself — same single point of failure remains.*

## 2. Horizontal Scaling

```mermaid
flowchart TD
    LB[Load Balancer] --> S1[Server 1]
    LB --> S2[Server 2]
    LB --> S3[Server 3]
    LB --> S4[Server 4]
    S1 --> DB[(Shared Database)]
    S2 --> DB
    S3 --> DB
    S4 --> DB
```
*Caption: Horizontal scaling adds more machines behind a load balancer, all sharing a common data store.*

## 3. Growth Path: Vertical First, Then Horizontal

```mermaid
flowchart LR
    A[Small App\nSingle Small Server] -->|Traffic grows| B[Vertical Scaling\nBigger Single Server]
    B -->|Hits ceiling / needs redundancy| C[Horizontal Scaling\nMultiple Servers + Load Balancer]
```
*Caption: A typical scaling journey — start with vertical scaling for simplicity, transition to horizontal scaling as growth demands it.*
