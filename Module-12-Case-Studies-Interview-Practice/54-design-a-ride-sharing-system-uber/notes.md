# Interview Cheat Sheet: Ride-Sharing System (Uber-like)

## Requirements

**Functional**
- Rider requests a ride (pickup + destination).
- Match rider to a nearby available driver.
- Real-time GPS tracking of driver (and rider) during approach and trip.
- Fare calculation, including surge pricing.
- Full trip lifecycle management + payment on completion.

**Non-functional**
- Low-latency matching (seconds, not tens of seconds).
- High availability (app must stay usable through partial failures).
- Geospatial scale (millions of moving drivers worldwide).
- Strong consistency for trip state (no double-booking) and payment (no double-charging); location data can be eventually consistent.

## Capacity Numbers (reference)

| Metric | Assumption | Result |
|---|---|---|
| Daily active riders | 20M | — |
| Daily active drivers | 5M (2M concurrently active at peak) | — |
| Location update interval | every 4 sec per active driver | 2M / 4 ≈ **500,000 location updates/sec** at peak |
| Rides/day | 20M riders × 1.2 rides/day | ≈ 24M rides/day |
| Ride requests/sec | 24M / 86,400 sec avg, ×3 peak factor | ≈ 280/sec avg, **~800-1,000/sec peak** |
| Trip record size | ~1 KB metadata + ~11 KB GPS trace (225 pings × 50 bytes) | ≈ 12 KB/trip |
| Trip history storage | 24M trips/day × 12 KB | ≈ **290 GB/day**, ≈ **100 TB/year** |

Takeaway: location writes (500K/sec) dominate over ride requests (~1K/sec) by two orders of magnitude — this is why location data needs a purpose-built, high-throughput, availability-favored store, not the same database as trip/payment records.

## Architecture Summary

Rider/Driver apps → Load Balancer → API Gateway →
- **Location Service** — ingests GPS pings, maintains geospatial index (geohash/quadtree), sharded via consistent hashing.
- **Matching Service** — queries Location Service for nearby available drivers, ranks candidates, assigns driver.
- **Trip Service** — owns the trip state machine (requested → assigned → arrived → in-progress → completed → paid); runs the trip as a Saga.
- **Payment Service** — fare calc, surge pricing, idempotent charge on trip completion.
- **Pub/Sub + WebSockets** — live location fan-out per trip topic, pushed to rider/driver apps over persistent WebSocket connections.

Datastores: geospatially-sharded in-memory store (e.g., Redis geo) for live locations; strongly-consistent relational/transactional store for trip + payment records; message broker for pub/sub and saga coordination; object/cold storage for historical GPS traces.

## Key Decisions & Trade-offs

| Decision | Options | Choice & Why |
|---|---|---|
| Geospatial index structure | Geohash vs. Quadtree | **Geohash** for simplicity, string-prefix locality, and easy use as a consistent-hashing key; **Quadtree** if driver density is very uneven and adaptive cell sizing is worth the added complexity. |
| Distributing location data across servers | Static range sharding vs. Consistent hashing | **Consistent hashing** over geohash prefixes — adding/removing location nodes reshuffles only a small fraction of geo-cells, minimizing rebalancing cost. |
| Coordinating trip + payment across services | 2PC vs. Saga | **Saga** — 2PC would hold cross-service locks for the whole trip duration (minutes); Saga uses local transactions + compensating actions (e.g., release driver reservation, retry/flag failed payment) and tolerates partial failure gracefully. |
| Preventing double-charge on retry | None vs. Idempotency key | **Idempotency key** derived from trip ID on every payment charge call — duplicate requests return the original result instead of charging twice. |
| Consistency model: location data | CP vs. AP | **AP (availability-favored)** — stale-by-a-second driver position is harmless; system must stay responsive under partition. |
| Consistency model: trip state / payment | CP vs. AP | **CP (consistency-favored)** — driver "reserved" flag and payment charge must be strongly consistent to avoid double-booking / double-charging. |
| Real-time client updates | Polling vs. Pub/Sub + WebSockets | **Pub/Sub + WebSockets** — push-based delivery avoids polling overhead and gives near-instant map updates. |
