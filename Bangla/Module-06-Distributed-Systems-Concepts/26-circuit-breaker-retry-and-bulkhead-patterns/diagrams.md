# ডায়াগ্রাম — Circuit Breaker, Retry & Bulkhead Patterns

## ১. Circuit Breaker State Machine

```mermaid
stateDiagram-v2
    [*] --> Closed

    Closed --> Open : failure rate exceeds threshold\n(e.g., >50% failures in rolling window)
    Closed --> Closed : request succeeds / failure counted

    Open --> HalfOpen : sleep window timeout expires\n(e.g., 30s elapsed)
    Open --> Open : calls fail immediately (fast fail, no request sent)

    HalfOpen --> Closed : probe requests meet\nsuccess threshold
    HalfOpen --> Open : probe request(s) fail

    Closed --> [*]
```

*Caption: Circuit breaker Closed অবস্থায় শুরু হয়, ধারাবাহিক failure-এ Open-এ trip করে, এবং সম্পূর্ণভাবে reset হওয়ার আগে Half-Open-এর মাধ্যমে সতর্কতার সাথে পুনরুদ্ধার probe করে।*

## ২. Exponential Backoff এবং Jitter সহ Retry

```mermaid
sequenceDiagram
    participant Client
    participant Service as Downstream Service

    Client->>Service: Attempt 1
    Service-->>Client: Failure (timeout)
    Note over Client: Wait ~100ms (base delay + jitter)

    Client->>Service: Attempt 2
    Service-->>Client: Failure (timeout)
    Note over Client: Wait ~200ms (2x base + jitter)

    Client->>Service: Attempt 3
    Service-->>Client: Failure (timeout)
    Note over Client: Wait ~400ms (4x base + jitter)

    Client->>Service: Attempt 4
    Service-->>Client: Success (200 OK)
    Note over Client: Retry budget respected — stop retrying
```

*Caption: প্রতিটি retry delay মোটামুটি দ্বিগুণ হয় এবং random jitter অন্তর্ভুক্ত করে, retry গুলো ছড়িয়ে দেয় যাতে অনেক client synchronized wave-এ service-কে আঘাত না করে।*

## ৩. বিভিন্ন Dependency জুড়ে Bulkhead Isolation

```mermaid
flowchart LR
    A[Incoming Requests] --> B[API Service]

    subgraph Bulkhead Pools
        direction TB
        P1["Thread Pool A\n(max 10 threads)\nfor Payment Service"]
        P2["Thread Pool B\n(max 10 threads)\nfor Inventory Service"]
        P3["Thread Pool C\n(max 10 threads)\nfor Recommendation Service"]
    end

    B --> P1 --> D1[(Payment Service - slow/failing)]
    B --> P2 --> D2[(Inventory Service - healthy)]
    B --> P3 --> D3[(Recommendation Service - healthy)]
```

*Caption: প্রতিটি downstream dependency তার নিজস্ব সীমিত thread pool পায়, তাই একটি ধীরগতির বা ব্যর্থ Payment Service শুধুমাত্র নিজের pool শেষ করে এবং কখনো Inventory বা Recommendations-এর call গুলোকে অনাহারে রাখে না।*
