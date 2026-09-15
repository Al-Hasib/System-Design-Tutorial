# Why This Topic Matters: Design a Ride-Sharing System (like Uber)

> **In one sentence:** This is the capstone because nothing else in the course combines all of it at once — continuous location writes at enormous volume, geospatial queries, a matching problem with real exclusivity constraints, real-time state across two live clients, and money.

## Why This Case Study Exists

Every other case study has one dominant characteristic: a URL shortener is read-heavy key-value, a feed is fan-out, streaming is bandwidth. Ride-sharing has several hard problems simultaneously, and they interact.

Drivers emit location updates every few seconds, continuously, whether or not anyone is looking — a write volume that dwarfs the number of actual rides by orders of magnitude. Riders need "who is near me right now," which is a query type that ordinary indexes cannot answer efficiently. Matching must assign one driver to one rider *exactly once*, which is a distributed exclusivity problem with real money attached. And once matched, both parties need live updates for the duration of the trip.

It is also the case study that most rewards saying "different data has different requirements," because within one product you have data that must be strongly consistent (payments, ride assignment) and data that should explicitly not be (a driver's location three seconds ago).

## The Design Problems It Forces You to Solve

### 1. Location writes at a volume that dominates everything
**The problem:** A million active drivers reporting every four seconds is a quarter of a million writes per second, continuously, most of which will never be read.

**Why it is hard:** Writing that into a general-purpose database with durability guarantees is both impossible at reasonable cost and unnecessary.

**What you learn:** To question the requirement. Locations are ephemeral — a position from thirty seconds ago is worthless — so they belong in memory (Redis) with a TTL, not in a durable store. Historical tracks, if needed for billing or dispute resolution, go through a queue into a time-series or columnar store asynchronously. Separating the hot path (current position, in memory) from the cold path (trip history, batched to durable storage) is the first and most important structural decision.

### 2. Finding nearby drivers efficiently
**The problem:** "Which drivers are within 3 km of this point?" cannot be answered by a B-tree on latitude and longitude — that index can narrow one dimension, not two.

**What you learn:** **Geospatial indexing** by reducing two dimensions to one. Geohash encodes a location as a string prefix where nearby points share prefixes, making a proximity query a prefix scan. Uber's H3 uses hexagonal cells (hexagons have uniform distance to all neighbors, unlike squares). Both make "nearby" a lookup into a small set of cells instead of a scan. You also learn the edge cases that make this real: a driver just across a cell boundary is nearby in reality but not in the index, so you query the surrounding ring of cells too; and cell size must be tuned, because dense downtown cells and sparse rural cells have very different occupancy.

### 3. Assigning one driver to exactly one rider
**The problem:** Three riders request simultaneously and the same nearby driver is the best match for all three. Each request is handled by a different server.

**Why it is hard:** This is a distributed mutual-exclusion problem, and getting it wrong means a driver receives two accepted rides.

**What you learn:** That the strongest solution is usually not a distributed lock but a **conditional atomic update** — claim the driver with an update that only succeeds if their status is still `available`, and let the database's own guarantee do the work. Where a lock is genuinely needed, it needs the fencing-token treatment from topic 40. You also learn the state machine around it: an offer with a short timeout, a fallback to the next candidate if declined or unanswered, and idempotent handling so a retried accept does not create two rides.

### 4. Real-time state for two parties at once
**The problem:** During a trip, the rider watches the driver approach and the driver receives navigation and status changes. Both need updates within seconds, for the whole trip.

**What you learn:** Persistent connections (WebSockets) for both parties, with the same cross-server routing problem as chat — a location update arriving at one server must reach a rider's socket held by another, which means a Pub/Sub layer and a session registry. You also learn to throttle deliberately: pushing every raw GPS update to the rider is wasteful, so you downsample and interpolate on the client. This is a good example of a product-quality decision that reduces infrastructure load.

### 5. Consistency requirements that differ within one product
**The problem:** Applying a single consistency standard makes the system either wrong or unaffordable.

**What you learn:** To classify explicitly. Payment and fare calculation must be strongly consistent and transactional. Ride assignment must be exclusive. Driver location can be seconds stale with no consequence. Surge pricing is computed from aggregated demand and is approximate by nature. Saying this out loud — "these three things need different guarantees, and here is why" — is the clearest demonstration of the judgment the whole course is building toward.

## What It Costs to Get Wrong

- **Storing every location update durably** produces a design that is orders of magnitude more expensive than it needs to be.
- **Indexing latitude and longitude conventionally** and scanning for proximity does not scale, and missing geospatial indexing entirely is the most common gap in this interview.
- **Ignoring double-assignment** leaves the core correctness problem of the product unaddressed.
- **Applying strong consistency everywhere** makes the location pipeline unbuildable.
- **Forgetting cell-boundary effects** in the geospatial index produces a matcher that misses the closest driver.

## Why Interviewers Choose This One

It is the broadest problem in common use, which makes it a natural capstone and a reliable way to probe depth in whichever direction the interviewer chooses. A candidate can be pushed toward the write pipeline, the geospatial index, the matching algorithm, the real-time layer, or the payment path — and each is a legitimate deep dive. It also rewards the two habits this course has been building throughout: estimate before designing, and choose guarantees per use case rather than per system.

## How It Connects

This case study brings together **geospatial indexing** on top of general **indexing** principles (topic 12), **Redis** for in-memory location state (topic 19), **message queues** for the asynchronous location and trip pipeline (topic 20), **stream processing** for surge pricing and demand aggregation (topic 23), **WebSockets** with **Pub/Sub** routing for live trip updates (topics 10, 21), **distributed locking** and **atomic conditional updates** for matching (topics 40, 37), **idempotency** for retried requests and payments (topic 29), **sharding** by geography (topic 14), **consistency models** applied per use case (topic 29), and **microservices** with saga-based flows for the ride-and-payment lifecycle (topics 30, 28).

**Where to go from here:** you have now walked the full path from "what is system design" to a system that exercises nearly every concept in it. The most useful next step is to redo two or three of these case studies from a blank page, out loud, with a timer — the gap between recognizing an argument and generating it under pressure is where interview performance actually lives.
