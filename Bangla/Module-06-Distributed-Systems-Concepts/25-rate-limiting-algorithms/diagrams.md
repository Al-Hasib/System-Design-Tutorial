# Diagrams: Rate Limiting Algorithms

## Token Bucket প্রবাহ (Token Bucket Flow)

```mermaid
flowchart TD
    A[Bucket: capacity = 100 tokens] -->|refills at 10 tokens/sec, up to capacity| A
    B[Request arrives] --> C{Token available in bucket?}
    C -->|Yes: consume 1 token| D[Request allowed / forwarded]
    C -->|No: bucket empty| E[Request rejected: 429 Too Many Requests]
```

*ক্যাপশন: Token গুলো bucket-এর capacity পর্যন্ত ক্রমাগত রিফিল হয়; প্রতিটি request একটি token consume করে, তাই নিষ্ক্রিয় সময় burst-কে token শেষ না হওয়া পর্যন্ত তাৎক্ষণিকভাবে পাস হতে দেয়।*

## Leaky Bucket প্রবাহ (Leaky Bucket Flow)

```mermaid
flowchart TD
    A[Request arrives] --> B{Queue / bucket has room?}
    B -->|Yes| C[Request added to bucket queue]
    B -->|No: bucket full| D[Request overflows / rejected]
    C --> E[Bucket leaks / processes requests at fixed constant rate]
    E --> F[Request forwarded to backend]
```

*ক্যাপশন: Request গুলো bucket-এ queue হয় এবং একটি কঠোরভাবে স্থির হারে ব্যাকএন্ডে drain হয়, আগত traffic যতই burst-প্রবণ হোক না কেন তা নির্বিশেষে।*

## Sliding Window Counter ধারণা (Sliding Window Counter Concept)

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

*ক্যাপশন: Sliding window counter পূর্ববর্তী এবং বর্তমান fixed-window count মিশ্রিত করে, পূর্ববর্তী window sliding view-এর সাথে কতটা এখনও ওভারল্যাপ করে তা দিয়ে ওজনযুক্ত করে, একটি প্রকৃত sliding log-কে সস্তায় আনুমানিক করে।*
