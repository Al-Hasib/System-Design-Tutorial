# Diagrams: CDN (Content Delivery Network) Explained

## ১. CDN Request Routing: Edge বনাম Origin

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

*Caption: Request-গুলো ভৌগোলিকভাবে নিকটতম edge server-এ route করা হয়; শুধুমাত্র cache miss-ই দূরবর্তী origin পর্যন্ত পুরোটা ভ্রমণ করে।*

## ২. নিকটতম PoP-তে Anycast Routing

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

*Caption: Anycast-এ, সব PoP একই IP address share করে; BGP routing স্বয়ংক্রিয়ভাবে প্রতিটি user-এর traffic-কে topologically নিকটতম PoP-তে পরিচালিত করে।*

## ৩. CDN Cache Lifecycle: TTL Expiration এবং Purge

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

*Caption: Edge server-গুলো cached content serve করতে থাকে যতক্ষণ না TTL শেষ হয় অথবা একটি explicit purge origin-এর বিরুদ্ধে revalidation বাধ্য করে।*
