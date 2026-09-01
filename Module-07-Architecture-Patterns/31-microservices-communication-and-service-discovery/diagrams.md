# Diagrams: Microservices Communication & Service Discovery

## 1. Synchronous vs Asynchronous Communication

```mermaid
flowchart TB
    subgraph Sync["Synchronous (REST/gRPC)"]
        A1[Order Service] -- "1. Check stock (blocks)" --> A2[Inventory Service]
        A2 -- "2. Response" --> A1
    end

    subgraph Async["Asynchronous (Event-Driven)"]
        B1[Order Service] -- "1. Publish OrderPlaced" --> B2[[Message Queue]]
        B2 --> B3[Notification Service]
        B2 --> B4[Analytics Service]
    end
```
*Synchronous calls block for an immediate answer; asynchronous events decouple the publisher from consumers in time.*

## 2. Client-Side vs Server-Side Service Discovery

```mermaid
flowchart LR
    subgraph ClientSide["Client-Side Discovery"]
        C1[Order Service] -- "1. Query registry" --> R1[(Service Registry)]
        R1 -- "2. List of healthy instances" --> C1
        C1 -- "3. Call chosen instance directly" --> C2[Inventory Instance]
    end

    subgraph ServerSide["Server-Side Discovery"]
        S1[Order Service] -- "1. Call stable endpoint" --> S2[Load Balancer / Router]
        S2 -- "2. Query registry" --> R2[(Service Registry)]
        S2 -- "3. Forward to healthy instance" --> S3[Inventory Instance]
    end
```
*Client-side discovery puts lookup and load balancing in the caller; server-side discovery hides it behind a load balancer.*

## 3. Service Mesh with Sidecar Proxies

```mermaid
flowchart LR
    subgraph OrderPod["Order Service Pod"]
        OApp[Order Service App] --> OSidecar[Sidecar Proxy]
    end
    subgraph InvPod["Inventory Service Pod"]
        ISidecar[Sidecar Proxy] --> IApp[Inventory Service App]
    end
    OSidecar <--> ISidecar
    OSidecar -.-> CP[Mesh Control Plane: Istio/Linkerd]
    ISidecar -.-> CP
```
*In a service mesh, sidecar proxies handle discovery, load balancing, retries, and security, while a control plane configures them all.*
