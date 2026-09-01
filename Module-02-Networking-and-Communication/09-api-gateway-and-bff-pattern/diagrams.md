# Diagrams: API Gateway & Backend-for-Frontend

## 1. API Gateway in Front of a Microservices Architecture

```mermaid
flowchart LR
    Mobile[Mobile Client] --> GW[API Gateway]
    Web[Web Client] --> GW
    Partner[Partner API Client] --> GW

    GW --> Auth[Auth Check]
    GW --> RL[Rate Limiter]
    GW --> R{Router}

    R --> U[User Service]
    R --> O[Order Service]
    R --> P[Payment Service]
    R --> Rec[Recommendation Service]
```
*A single gateway centralizes authentication and rate limiting for every client before routing requests to the correct backend microservice.*

## 2. Request Aggregation at the Gateway

```mermaid
sequenceDiagram
    participant M as Mobile Client
    participant GW as API Gateway
    participant U as User Service
    participant P as Posts Service
    participant SG as Social Graph Service

    M->>GW: GET /profile-summary
    GW->>U: GET /users/42
    GW->>P: GET /users/42/recent-posts
    GW->>SG: GET /users/42/follow-counts
    U-->>GW: user data
    P-->>GW: recent posts
    SG-->>GW: follower/following counts
    GW-->>M: 200 OK (single aggregated response)
```
*The gateway fans out one client request into three fast internal calls, sparing the mobile client three separate slow round trips.*

## 3. API Gateway + Backend-for-Frontend Layers

```mermaid
flowchart TB
    Mobile[Mobile Client] --> GW[API Gateway - TLS + top-level Auth]
    Web[Web Client] --> GW

    GW --> MBFF[Mobile BFF]
    GW --> WBFF[Web BFF]

    MBFF --> Svc1[User Service]
    MBFF --> Svc2[Posts Service]
    WBFF --> Svc1
    WBFF --> Svc2
    WBFF --> Svc3[Analytics Service]
```
*The gateway handles universal concerns for every client, then routes to a client-specific BFF that shapes and aggregates data exactly the way that client needs.*
