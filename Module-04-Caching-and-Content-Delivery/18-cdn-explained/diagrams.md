# Diagrams: CDN (Content Delivery Network) Explained

## 1. CDN Request Routing: Edge vs Origin

```mermaid
flowchart TD
    U[User in Singapore] -->|Request content| E1[Nearest Edge Server - Singapore PoP]
    E1 -->|Cache hit| U
    E1 -->|Cache miss| O[Origin Server - Virginia, USA]
    O -->|Return content| E1
    E1 -->|Cache the content and serve| U

    U2[User in Germany] -->|Request content| E2[Nearest Edge Server - Frankfurt PoP]
    E2 -->|Cache hit| U2
    E2 -->|Cache miss| O
```

*Caption: Requests are routed to the geographically nearest edge server; only cache misses travel all the way back to the distant origin.*

## 2. Anycast Routing to the Nearest PoP

```mermaid
flowchart LR
    subgraph Backbone["Internet Backbone (BGP Routing)"]
        direction TB
        R[Same Anycast IP announced from multiple PoPs]
    end
    U1[User - Tokyo] --> R
    U2[User - London] --> R
    U3[User - New York] --> R
    R --> P1[PoP - Tokyo]
    R --> P2[PoP - London]
    R --> P3[PoP - New York]
```

*Caption: With Anycast, all PoPs share the same IP address; BGP routing automatically directs each user's traffic to the topologically nearest PoP.*

## 3. CDN Cache Lifecycle: TTL Expiration and Purge

```mermaid
sequenceDiagram
    participant User
    participant Edge as Edge Server
    participant Origin as Origin Server

    User->>Edge: GET /image.png
    Edge->>Origin: Fetch (cache miss)
    Origin-->>Edge: 200 OK + Cache-Control max-age=3600
    Edge-->>User: Serve content, cache locally

    Note over Edge: Within TTL window
    User->>Edge: GET /image.png (again)
    Edge-->>User: Serve from cache (cache hit)

    Note over Origin,Edge: Admin issues manual purge
    Origin->>Edge: Purge /image.png
    User->>Edge: GET /image.png
    Edge->>Origin: Fetch (cache miss, forced by purge)
    Origin-->>Edge: 200 OK (new version)
    Edge-->>User: Serve fresh content
```

*Caption: Edge servers serve cached content until the TTL expires or an explicit purge forces revalidation against the origin.*
