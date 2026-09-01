# Diagrams: URL Shortener

## 1. High-Level Architecture

```mermaid
flowchart LR
    Client[Client Browser / App]
    LB[Load Balancer]
    App1[App Server 1]
    App2[App Server 2]
    App3[App Server N]
    KGS[Key Generation Service]
    Cache[(Cache - Cache-Aside\nConsistent Hashing)]
    DB[(Database Shards\nConsistent Hashing)]

    Client --> LB
    LB --> App1
    LB --> App2
    LB --> App3
    App1 --> KGS
    App2 --> KGS
    App3 --> KGS
    App1 --> Cache
    App2 --> Cache
    App3 --> Cache
    Cache --> DB
    App1 -.fallback on cache miss.-> DB
    App2 -.fallback on cache miss.-> DB
    App3 -.fallback on cache miss.-> DB
```

*Caption: Client requests hit a load balancer, fan out across stateless app servers, which consult a key generation service on writes and a cache-aside layer in front of sharded databases on reads.*

## 2. Write Path (Shorten a URL)

```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant A as App Server
    participant K as Key Generation Service
    participant Ca as Cache
    participant D as Database (Sharded)

    C->>LB: POST /shorten { longUrl }
    LB->>A: Route request
    A->>K: Request unique short code
    K-->>A: shortCode (base62)
    A->>D: Write { shortCode -> longUrl }
    A->>Ca: Populate cache { shortCode -> longUrl }
    A-->>C: 201 Created { shortUrl }
```

*Caption: On write, the app server obtains a collision-free short code from the key generation service, persists the mapping to the sharded database, and warms the cache before responding.*

## 3. Read Path (Redirect)

```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant A as App Server
    participant Ca as Cache
    participant D as Database (Sharded)

    C->>LB: GET /{shortCode}
    LB->>A: Route request
    A->>Ca: Lookup shortCode
    alt Cache hit
        Ca-->>A: longUrl
    else Cache miss
        A->>D: Lookup shortCode
        D-->>A: longUrl
        A->>Ca: Populate cache (cache-aside)
        Note over A,D: Async: log click event for analytics
    end
    A-->>C: 301/302 Redirect to longUrl
```

*Caption: The read path checks the cache-aside layer first for near-instant redirects, falling back to the sharded database only on a cache miss, with click logging kept off the critical path.*
