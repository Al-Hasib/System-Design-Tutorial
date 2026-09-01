# Diagrams — Circuit Breaker, Retry & Bulkhead Patterns

## 1. Circuit Breaker State Machine

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

Caption: The circuit breaker starts Closed, trips to Open on sustained failures, and cautiously probes recovery via Half-Open before fully resetting.

## 2. Retry with Exponential Backoff and Jitter

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
    Note over Client: Retry budget respected; stop retrying
```

Caption: Each retry delay roughly doubles and includes random jitter, spreading out retries so many clients don't hammer the service in synchronized waves.

## 3. Bulkhead Isolation Across Dependencies

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

Caption: Each downstream dependency gets its own bounded thread pool, so a slow or failing Payment Service exhausts only its own pool and never starves calls to Inventory or Recommendations.
