# Diagrams: Microservices Communication ও Service Discovery

## ১. Synchronous বনাম Asynchronous Communication

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
*Synchronous call তাৎক্ষণিক উত্তরের জন্য block করে; asynchronous event publisher-কে consumer থেকে সময়ের দিক থেকে decouple করে।*

## ২. Client-Side বনাম Server-Side Service Discovery

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
*Client-side discovery lookup ও load balancing caller-এর মধ্যে রাখে; server-side discovery সেটা একটা load balancer-এর পেছনে লুকিয়ে রাখে।*

## ৩. Sidecar Proxy-সহ Service Mesh

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
*একটি service mesh-এ, sidecar proxy discovery, load balancing, retries, এবং security সামলায়, আর একটা control plane এগুলো সবকিছু configure করে।*
