# Diagrams: Rate Limiting Algorithms

## Token Bucket Flow

```mermaid
flowchart TD
    A[Bucket: capacity = 100 tokens] -->|refills at 10 tokens/sec, up to capacity| A
    B[Request arrives] --> C{Token available in bucket?}
    C -->|Yes: consume 1 token| D[Request allowed / forwarded]
    C -->|No: bucket empty| E[Request rejected: 429 Too Many Requests]
```

*Caption: Tokens refill continuously up to the bucket's capacity; each request consumes one token, so idle time lets bursts through instantly until tokens run out.*

## Leaky Bucket Flow

```mermaid
flowchart TD
    A[Request arrives] --> B{Queue / bucket has room?}
    B -->|Yes| C[Request added to bucket queue]
    B -->|No: bucket full| D[Request overflows / rejected]
    C --> E[Bucket leaks / processes requests at fixed constant rate]
    E --> F[Request forwarded to backend]
```

*Caption: Requests queue up in the bucket and are drained to the backend at a strictly constant rate, regardless of how bursty the incoming traffic was.*

## Sliding Window Counter Concept

```mermaid
sequenceDiagram
    participant Client
    participant Limiter as Rate Limiter (Redis)
    participant Backend

    Client->>Limiter: Request at t (mid previous+current window)
    Limiter->>Limiter: weighted_count = current_window_count + previous_window_count * overlap_fraction
    alt weighted_count < limit
        Limiter->>Backend: forward request
        Limiter->>Limiter: increment current_window_count
    else weighted_count >= limit
        Limiter-->>Client: 429 Too Many Requests
    end
```

*Caption: The sliding window counter blends the previous and current fixed-window counts, weighted by how much of the previous window still overlaps the sliding view, approximating a true sliding log cheaply.*
