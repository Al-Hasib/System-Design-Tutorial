# Diagrams: API Gateway ও Backend-for-Frontend

## ১. একটি Microservices আর্কিটেকচারের সামনে API Gateway

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
*সঠিক backend microservice-এ request route করার আগে একটি একক gateway প্রতিটি client-এর জন্য authentication এবং rate limiting কেন্দ্রীভূত করে।*

## ২. Gateway-তে Request Aggregation

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
*Gateway একটি client request-কে তিনটি দ্রুত internal call-এ fan out করে, mobile client-কে তিনটি আলাদা ধীর round trip থেকে রক্ষা করে।*

## ৩. API Gateway + Backend-for-Frontend Layer

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
*Gateway প্রতিটি client-এর জন্য universal concern গুলো handle করে, তারপর একটি client-specific BFF-এ route করে যা সেই client-এর ঠিক প্রয়োজন অনুযায়ী data shape ও aggregate করে।*
