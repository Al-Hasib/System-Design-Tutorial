# Diagrams — Design a Rate Limiter

Companion diagrams for [`README.md`](./README.md). Terminology matches `notes.md`.

## 1. High-Level Architecture

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

*Caption: Requests are checked against a centralized, sharded Redis store at the API gateway; rejected requests short-circuit at the edge and never reach downstream services.*

## 2. Request-Check Sequence (Token Bucket via Redis Lua Script)

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

*Caption: The check-and-decrement happens as a single atomic Lua script on Redis, eliminating the check-then-increment race condition across many concurrent gateway instances.*

## 3. Failure Mode — Fail-Open vs Fail-Closed When Redis Is Unavailable

```mermaid
flowchart TD
    Req[Incoming request] --> CB{Circuit breaker:<br/>Redis reachable?}
    CB -- yes --> Normal["Normal atomic<br/>token-bucket check"]
    CB -- no / tripped --> Policy{Endpoint sensitivity}
    Policy -- "public / read-heavy" --> FailOpen["Fail-open:<br/>allow with local fallback limit"]
    Policy -- "login / password-reset" --> FailClosed["Fail-closed:<br/>reject request"]
```

*Caption: When the rate-limiter store itself is down, a circuit breaker routes to an explicit fail-open or fail-closed policy chosen per endpoint's sensitivity.*
