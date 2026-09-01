# Design a Ride-Sharing System (like Uber)

**Difficulty:** Advanced (Capstone). **Estimated length:** 25-30 min. **Prerequisites:** [WebSockets, Long Polling, and SSE](../../Module-02-Networking-and-Communication/10-websockets-long-polling-and-sse/README.md), [Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md), [CAP Theorem and PACELC](../../Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md), [Publish-Subscribe Pattern](../../Module-05-Messaging-and-Asynchronous-Systems/21-publish-subscribe-pattern/README.md), [Consistent Hashing Explained](../../Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/README.md), [Distributed Transactions: 2PC and Saga](../../Module-06-Distributed-Systems-Concepts/28-distributed-transactions-2pc-and-saga/README.md), [Data Consistency Models and Idempotency](../../Module-06-Distributed-Systems-Concepts/29-data-consistency-models-and-idempotency/README.md)

## Learning Objectives

- Frame a ride-sharing platform as a set of cooperating services: location, matching, trip lifecycle, and payment.
- Estimate capacity for a geospatially-distributed, real-time system with continuous location streams.
- Apply geohashing/quadtrees together with consistent hashing to shard and query location data at scale.
- Model a multi-step, multi-service trip flow as a Saga with compensating actions and idempotent payment calls.
- Reason about CAP/PACELC trade-offs differently for location data (availability-favored) versus payment data (consistency-favored).

## Script

### Hook / Intro

Picture this: it's 11 p.m., a concert just let out, and three thousand people open their ride-sharing app within the same two minutes, all standing within a quarter-mile radius. The system has to find them a driver in seconds, track that driver's exact position as they weave through traffic, calculate a fare that reflects surging demand, and charge a card at the end — without double-booking a single driver or double-charging a single rider. That's the ride-sharing system design problem, and it's one of the best interview questions there is because it forces you to combine geospatial indexing, real-time messaging, distributed transactions, and consistency trade-offs into one coherent design. Today we're designing a system like Uber or Lyft from scratch. Let's clarify requirements first.

### Step 1: Clarify Requirements

Before drawing boxes, we nail down scope with the interviewer.

**Functional requirements:**
- A rider can request a ride by specifying a pickup location and destination.
- The system matches the rider with a nearby available driver.
- Both rider and driver see each other's live GPS location on a map during the approach and the trip.
- The system calculates a fare based on distance, time, and demand (surge pricing).
- The system manages the full trip lifecycle — requested, driver assigned, driver arrived, trip started, trip completed — and processes payment at the end.

**Non-functional requirements:**
- **Low-latency matching**: a rider should get a driver match within a few seconds, not tens of seconds.
- **High availability**: the app must stay usable even during regional infrastructure hiccups — a stuck request loses trust and revenue immediately.
- **Geospatial scale**: the system must efficiently answer "who is near me" queries across millions of moving drivers worldwide.
- **Consistency where it matters**: trip state (a driver can't be double-booked) and payment (a rider can't be double-charged, and a completed trip must eventually be paid for) need strong consistency guarantees, even though location pings can tolerate slight staleness.

We explicitly call out that we're favoring availability for location data and consistency for trip-state and payment data — that split drives most of the interesting decisions later.

### Step 2: Capacity Estimation

Let's ground this in real numbers. Say we have **20 million daily active riders** and **5 million daily active drivers**, of which roughly **2 million drivers are concurrently active** during peak hours.

**Location updates:** each active driver's app pushes a GPS ping every 4 seconds. That's:

2,000,000 drivers / 4 seconds ≈ **500,000 location updates per second** at peak.

This is the dominant write load in the whole system — far more than ride requests — and it tells us immediately that the location pipeline needs to be built for high-throughput, low-durability-requirement writes, not a traditional ACID database.

**Ride requests:** assume 20 million riders average 1.2 rides/day, giving roughly 24 million rides/day. Averaged over 86,400 seconds, that's about 280 requests/second, but ride demand is bursty — rush hour, concerts, bad weather — so we plan for a **peak of roughly 800-1,000 ride requests per second**.

**Trip history storage:** each trip has metadata (~1 KB: rider ID, driver ID, timestamps, fare, status) plus a GPS trace. A 15-minute trip pinging every 4 seconds produces about 225 points at ~50 bytes each, or ~11 KB of trace data. That's roughly 12 KB/trip, and at 24 million trips/day that's about **290 GB/day**, or on the order of **100 TB/year** — very manageable for cold object storage with a hot/warm tiering strategy.

These numbers justify two design instincts: (1) location data needs an in-memory, horizontally-sharded store, not a relational database, and (2) trip and payment records are comparatively low-volume and can afford stronger consistency mechanisms.

### Step 3: High-Level Design

At a high level, requests from rider and driver mobile apps hit a **load balancer**, then an **API gateway** that handles auth, rate limiting, and routing. From there:

- A **Location Service** ingests the 500K/sec driver GPS pings and maintains a geospatial index of "where is every driver right now."
- A **Matching Service** takes a ride request, queries the Location Service for nearby available drivers, scores and ranks candidates (distance, ETA, driver rating), and assigns one.
- A **Trip Service** owns the trip state machine (requested → assigned → arrived → in-progress → completed → paid) and is the source of truth for trip status.
- A **Payment Service** handles fare calculation, surge pricing, and charging the rider's stored payment method at trip completion.
- Live position updates flow to both apps via a **pub/sub layer** feeding **WebSocket** connections, so the rider watches the driver icon move on the map in near real time without polling.

Underneath, we'd use a geospatially-partitioned key-value store (e.g., Redis with geo commands, or a custom geohash-sharded store) for live driver locations, a relational or strongly-consistent store for trip and payment records, and a message broker for the pub/sub fan-out and for inter-service coordination during the trip saga.

### Step 4: Deep Dive on Key Components

**Geospatial indexing and consistent hashing.** To answer "find available drivers within 2 km of this point" quickly across millions of drivers, we can't scan every record. We use **geohashing** (or a **quadtree**) to convert lat/long into a string or tree path that clusters nearby points together — a shared geohash prefix means physical proximity. That solves the query problem, but we also need to solve the *distribution* problem: which server holds the geohash bucket for downtown Chicago? This is exactly where **consistent hashing** from Module 6 comes in — we hash geohash prefixes (or use them directly as hash-ring keys) to assign location shards to nodes, so adding or removing a location-service node only reshuffles a small fraction of geographic cells instead of the entire dataset. Geohash gives us locality for querying; consistent hashing gives us locality-preserving, low-churn distribution across the fleet of servers holding that data.

**Real-time location broadcast.** Once a trip is matched, both the rider and driver apps need a live feed of the driver's position. The driver's app publishes each GPS ping to a **pub/sub** topic scoped to that trip (as covered in Module 5's publish-subscribe pattern) — the Location Service or a lightweight relay publishes, and any subscriber interested in that trip ID receives it. Each rider and driver app maintains a persistent **WebSocket** connection (Module 2) to a gateway that's subscribed to their active trip's topic, so updates push instantly instead of the client polling an HTTP endpoint every few seconds. This combination — pub/sub for fan-out, WebSockets for delivery — is what makes the "watch your driver approach" experience feel live.

**Trip lifecycle as a Saga.** A single ride touches multiple services: reserve a driver, confirm the ride with both parties, run the trip, and charge a payment — and these are separate services with separate datastores, so a classic ACID transaction spanning all of them isn't practical, and two-phase commit (Module 6) would hold locks across services for the entire trip duration, which is untenable. Instead we model the trip as a **Saga**: a sequence of local transactions, each with a defined **compensating action** if a later step fails. Step 1: tentatively reserve the driver (mark them "pending," not fully booked). Step 2: confirm the match with both apps. Step 3: run the trip and mark it completed. Step 4: charge payment. If payment fails at step 4, the compensating action might be to retry with a backup payment method, then flag the trip for collections rather than trying to "undo" a physical trip that already happened — compensation in a saga is about business-level rollback, not literal reversal. If the driver reservation in step 1 fails or times out, we simply release the driver and re-run matching — no downstream steps have happened yet, so there's nothing to compensate. Crucially, the payment charge call carries an **idempotency key** derived from the trip ID (Module 6's consistency/idempotency concepts) — if a network blip causes the Payment Service to receive the same "charge this trip" request twice, the second call is recognized as a duplicate and returns the original result instead of charging the rider twice.

### Step 5: Bottlenecks & Trade-offs

A few tensions are worth surfacing explicitly, because a good candidate names trade-offs instead of pretending there's a perfect answer.

**Matching accuracy vs. latency.** We could search a huge radius and deeply score every candidate driver for the objectively best match, but that adds latency the rider feels. In practice we search a small radius first, expand only if no driver is found, and use lightweight heuristics (distance, ETA) rather than exhaustive optimization — "good enough, fast" beats "optimal, slow" for this UX.

**Hot zones and surge pricing.** When a concert lets out, demand in one small geohash cell spikes far beyond local driver supply. The Matching Service should detect this imbalance (open ride requests vastly outnumbering available drivers in a cell) and the Payment Service applies **surge pricing** — both to ration scarce supply through price and to incentivize nearby drivers to reposition toward the hot zone.

**Consistency of driver availability under concurrency.** Two ride requests can try to match the same driver at nearly the same instant. This is a classic race condition, and it's exactly why the Saga's first step is a conditional, atomic "reserve" operation (e.g., a compare-and-set on driver status) rather than an optimistic assignment — we need strong consistency for that one flag, even in a system that's otherwise built for high throughput.

**CAP trade-offs, applied differently per subsystem.** Location data is high-volume, frequently overwritten, and tolerant of staleness — a driver pin being half a second old is harmless — so we favor **availability and partition tolerance** (AP) there, accepting eventual consistency. Payment and trip-state data is low-volume but must never be wrong — we favor **consistency** (CP) there, accepting that a node partition might briefly reject a write rather than risk a double-charge or double-booking. This is the core CAP/PACELC lesson from Module 3: the right choice isn't universal, it's per-subsystem, based on what an inconsistency would actually cost you.

### Recap

We designed a ride-sharing platform by splitting concerns into a Location Service built for massive-throughput, availability-favored writes using geohashing plus consistent hashing; a Matching Service balancing latency against match quality; a Trip Service running the lifecycle as a Saga with compensating actions; and a Payment Service protected by idempotency keys. Real-time tracking rides on pub/sub and WebSockets. And throughout, we made deliberate, subsystem-specific CAP trade-offs rather than a single blanket consistency model.

### What's Next

This case study leaned heavily on geospatial indexing, pub/sub, sagas, and CAP trade-offs — if any of those felt like a refresher rather than new material, that's the point: capstone case studies exist to show how the individual building blocks from earlier modules combine under real interview pressure. Try sketching this design yourself before your next mock interview, and compare your fare-calculation and surge-pricing approach against ours.

## Key Takeaways

- Split the system by concern: Location Service, Matching Service, Trip Service (state machine), and Payment Service, each with its own consistency needs.
- Location data is high-throughput and staleness-tolerant — favor availability (AP); trip state and payment data must be correct — favor consistency (CP).
- Geohashing/quadtrees give locality for "nearby driver" queries; consistent hashing gives low-churn distribution of that geospatial data across servers.
- Pub/sub plus WebSockets deliver live location tracking without client-side polling.
- Model the multi-service trip flow as a Saga with compensating actions, not two-phase commit — and protect payment calls with idempotency keys to prevent double-charging.
- Name trade-offs explicitly in an interview: matching speed vs. accuracy, and per-subsystem CAP choices, rather than claiming one design is "perfect."
