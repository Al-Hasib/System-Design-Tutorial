# Diagrams — Design a Rate Limiter

[`README.md`](./README.md)-এর সহায়ক diagram। Terminology `notes.md`-এর সাথে মিলে যায়।

## ১. High-Level Architecture

```mermaid
flowchart LR
    Client([Client])
    LB[Load Balancer]
    GW["API Gateway<br/>(rate-limiter middleware)"]
    Redis[("Sharded Redis Cluster<br/>(counters + Lua scripts)")]
    Svc[Downstream Microservices]

    Client --> LB --> GW
    GW <--> Redis
    GW -- under limit --> Svc
    GW -- over limit --> Reject([429 Too Many Requests<br/>+ Retry-After])
```

*Caption: Request গুলো API gateway-তে একটি কেন্দ্রীভূত, sharded Redis store-এর বিরুদ্ধে check করা হয়; reject করা request গুলো edge-এ short-circuit হয় এবং কখনো downstream service-এ পৌঁছায় না।*

## ২. Request-Check Sequence (Redis Lua Script-এর মাধ্যমে Token Bucket)

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant R as Redis (Lua EVAL)
    participant S as Downstream Service

    C->>GW: HTTP request (user/API key identity)
    GW->>R: EVAL token_bucket_check(key, capacity, refill_rate)
    Note over R: Atomically read, refill, and decrement token count in one round trip
    alt token available
        R-->>GW: allowed, tokens_remaining
        GW->>S: forward request
        S-->>GW: response
        GW-->>C: 200 OK
    else no token available
        R-->>GW: denied
        GW-->>C: 429 Too Many Requests + Retry-After
    end
```

*Caption: check-and-decrement একটি একক atomic Lua script হিসেবে Redis-এ ঘটে, যা অনেক concurrent gateway instance জুড়ে check-then-increment race condition দূর করে।*

## ৩. Failure Mode — Redis Unavailable থাকলে Fail-Open বনাম Fail-Closed

```mermaid
flowchart TD
    Req[Incoming request] --> CB{Circuit breaker:<br/>Redis reachable?}
    CB -- yes --> Normal["Normal atomic<br/>token-bucket check"]
    CB -- no / tripped --> Policy{Endpoint sensitivity}
    Policy -- "public / read-heavy" --> FailOpen["Fail-open:<br/>allow with local fallback limit"]
    Policy -- "login / password-reset" --> FailClosed["Fail-closed:<br/>reject request"]
```

*Caption: যখন rate-limiter store নিজেই বন্ধ থাকে, একটি circuit breaker প্রতিটি endpoint-এর sensitivity অনুযায়ী নির্বাচিত একটি স্পষ্ট fail-open বা fail-closed policy-তে route করে।*
