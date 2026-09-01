# Diagrams: Forward Proxy vs Reverse Proxy

## 1. Forward Proxy — Protecting the Client

```mermaid
flowchart LR
    C1[Employee Laptop A] --> FP[Forward Proxy]
    C2[Employee Laptop B] --> FP
    FP --> Internet((Internet))
    Internet --> Server[Destination Website]

    Server -.->|"sees only the proxy's IP"| FP
```
*The destination server only ever sees the forward proxy's identity — it has no visibility into which specific client made the request.*

## 2. Reverse Proxy — Protecting the Server

```mermaid
flowchart LR
    Client[Customer Browser] --> RP[Reverse Proxy]
    RP --> S1[Backend Server 1]
    RP --> S2[Backend Server 2]
    RP --> S3[Backend Server 3]

    Client -.->|"sees only the reverse proxy"| RP
```
*The client believes it's talking directly to "the server" — it never learns that a fleet of backend servers exists behind the reverse proxy.*

## 3. Side-by-Side: Same Traffic Path, Opposite Purpose

```mermaid
flowchart TB
    subgraph Forward["Forward Proxy Flow"]
        direction LR
        FC[Client] --> FProxy[Forward Proxy] --> FS[Server]
    end
    subgraph Reverse["Reverse Proxy Flow"]
        direction LR
        RC[Client] --> RProxy[Reverse Proxy] --> RS[Backend Servers]
    end
```
*Structurally identical — client, proxy, server — but the forward proxy is deployed by and represents the client side, while the reverse proxy is deployed by and represents the server side.*
