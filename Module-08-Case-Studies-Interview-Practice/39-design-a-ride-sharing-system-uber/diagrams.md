# Diagrams: Ride-Sharing System (Uber-like)

## 1. Overall Architecture

```mermaid
flowchart TB
    RiderApp["Rider App"]
    DriverApp["Driver App"]

    LB["Load Balancer"]
    GW["API Gateway"]

    LocSvc["Location Service<br/>(geohash/quadtree index,<br/>consistent-hash sharded)"]
    MatchSvc["Matching Service"]
    TripSvc["Trip Service<br/>(trip state machine / Saga)"]
    PaySvc["Payment Service<br/>(idempotent charge)"]

    PubSub["Pub/Sub Layer"]
    WS["WebSocket Gateway"]

    LocCache["Geo-sharded In-Memory Store<br/>(live driver locations)"]
    TripDB["Trip / Payment Store<br/>(strongly consistent)"]
    ColdStore["Cold Storage<br/>(historical GPS traces)"]

    RiderApp --> LB
    DriverApp --> LB
    LB --> GW

    GW --> LocSvc
    GW --> MatchSvc
    GW --> TripSvc
    GW --> PaySvc

    LocSvc <--> LocCache
    LocSvc --> ColdStore
    MatchSvc --> LocSvc
    MatchSvc --> TripSvc
    TripSvc --> PaySvc
    TripSvc <--> TripDB
    PaySvc <--> TripDB

    LocSvc --> PubSub
    PubSub --> WS
    WS --> RiderApp
    WS --> DriverApp
```

*Rider and driver apps reach the backend through a load balancer and API gateway; the Location Service (geohash + consistent hashing) feeds live positions to both a geo-sharded cache and a pub/sub layer that pushes real-time updates over WebSockets, while the Trip and Payment services keep trip/payment state in a strongly-consistent store.*

## 2. Ride Request → Match → Trip → Payment Saga (with Compensation)

```mermaid
sequenceDiagram
    participant Rider
    participant MatchSvc as Matching Service
    participant LocSvc as Location Service
    participant TripSvc as Trip Service
    participant Driver
    participant PaySvc as Payment Service

    Rider->>MatchSvc: Request ride (pickup, destination)
    MatchSvc->>LocSvc: Query nearby available drivers
    LocSvc-->>MatchSvc: Candidate driver list

    MatchSvc->>TripSvc: Reserve driver (conditional CAS)
    alt Driver reservation succeeds
        TripSvc-->>MatchSvc: Reserved
        MatchSvc->>Driver: Send ride offer
        Driver-->>MatchSvc: Accept
        MatchSvc->>TripSvc: Confirm match (state: assigned)
        TripSvc->>Rider: Driver assigned + live tracking (pub/sub)
        TripSvc->>Driver: Trip started (state: in-progress)
        Driver->>TripSvc: Trip completed (state: completed)
        TripSvc->>PaySvc: Charge fare (idempotency key = trip ID)
        alt Payment succeeds
            PaySvc-->>TripSvc: Payment confirmed
            TripSvc-->>Rider: Trip closed, receipt sent
        else Payment fails
            PaySvc-->>TripSvc: Payment failed
            TripSvc->>PaySvc: Retry with backup payment method
            alt Retry succeeds
                PaySvc-->>TripSvc: Payment confirmed
                TripSvc-->>Rider: Trip closed, receipt sent
            else Retry fails (compensation)
                TripSvc->>TripSvc: Flag trip for manual collection
                TripSvc-->>Rider: Trip closed, payment pending
            end
        end
    else Driver reservation fails (already taken)
        TripSvc-->>MatchSvc: Reservation denied
        MatchSvc->>LocSvc: Re-query next candidate
    end
```

*The trip is coordinated as a Saga of local transactions — reserve driver, confirm match, run trip, charge payment — with a compensating retry/flag-for-collection path if payment fails, and a re-match path if the initial driver reservation is denied.*
