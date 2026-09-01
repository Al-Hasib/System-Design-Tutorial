# Diagrams: Monolith vs Microservices

## 1. Monolithic Architecture

```mermaid
flowchart TB
    Client[Client Apps] --> LB[Load Balancer]
    LB --> M1[Monolith Instance 1]
    LB --> M2[Monolith Instance 2]

    subgraph "Monolith Process"
        direction TB
        UI[Presentation Layer]
        BL[Business Logic: Users, Orders, Payments, Notifications]
        DAL[Data Access Layer]
        UI --> BL --> DAL
    end

    M1 -.-> UI
    M2 -.-> UI
    DAL --> DB[(Single Shared Database)]
```
*A monolith deploys all modules as one unit, replicated behind a load balancer, sharing one database.*

## 2. Microservices Architecture

```mermaid
flowchart LR
    Client[Client Apps] --> GW[API Gateway]
    GW --> UserSvc[User Service]
    GW --> OrderSvc[Order Service]
    GW --> PaySvc[Payment Service]
    GW --> NotifSvc[Notification Service]

    UserSvc --> UserDB[(User DB)]
    OrderSvc --> OrderDB[(Order DB)]
    PaySvc --> PayDB[(Payment DB)]

    OrderSvc -- async event --> Queue[[Message Queue]]
    Queue --> NotifSvc
```
*Each microservice is independently deployable, owns its own database, and communicates over the network directly or via a message queue.*

## 3. Strangler Fig Migration Path

```mermaid
flowchart LR
    Client[Client Apps] --> GW[API Gateway / Facade]
    GW -- legacy routes --> Mono[Remaining Monolith]
    GW -- new route: /payments --> PaySvc[New Payment Service]
    GW -- new route: /notifications --> NotifSvc[New Notification Service]
    Mono --> DB[(Legacy Database)]
    PaySvc --> PayDB[(Payment DB)]
    NotifSvc --> NotifDB[(Notification DB)]
```
*The strangler fig pattern routes traffic for newly-extracted capabilities to new services while the shrinking monolith still serves everything else.*
