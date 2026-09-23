# Diagrams: Ride-Sharing System (Uber-এর মতো)

## ১. সামগ্রিক আর্কিটেকচার (Overall Architecture)

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

*Rider এবং driver app একটি load balancer এবং API gateway-এর মাধ্যমে backend-এ পৌঁছায়; Location Service (geohash + consistent hashing) live position একটি geo-sharded cache এবং একটি pub/sub layer উভয়কেই ফিড করে, যা WebSocket-এর মাধ্যমে real-time update push করে, এবং একই সময়ে Trip ও Payment service trip/payment state একটি strongly-consistent store-এ রাখে।*

## ২. Ride Request → Match → Trip → Payment Saga (Compensation সহ)

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

*Trip-টিকে local transaction-এর একটি Saga হিসেবে সমন্বয় করা হয় — driver সংরক্ষণ, match নিশ্চিতকরণ, trip পরিচালনা, payment charge — payment ব্যর্থ হলে একটি compensating retry/flag-for-collection path সহ, এবং প্রাথমিক driver reservation প্রত্যাখ্যাত হলে একটি re-match path সহ।*
