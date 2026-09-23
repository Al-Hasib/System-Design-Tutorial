# ডায়াগ্রাম: URL Shortener

## ১. High-Level Architecture

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

*Caption: Client request একটি load balancer-এ hit করে, stateless app server জুড়ে fan out হয়, যেগুলো write-এ একটি key generation service-এর এবং read-এ sharded database-এর সামনে একটি cache-aside layer-এর সাথে পরামর্শ করে।*

## ২. Write Path (একটি URL Shorten করা)

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

*Caption: Write-এ, app server key generation service থেকে একটি collision-free short code পায়, sharded database-এ mapping persist করে, এবং response দেওয়ার আগে cache warm করে।*

## ৩. Read Path (Redirect)

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

*Caption: Read path প্রথমে near-instant redirect-এর জন্য cache-aside layer check করে, শুধুমাত্র cache miss হলে sharded database-এ fall back করে, এবং click logging-কে critical path থেকে দূরে রাখে।*
