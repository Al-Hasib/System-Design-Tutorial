# Follow-Up Interview Questions

**1. How would you handle surge pricing?**
Continuously track the ratio of open ride requests to available drivers per geohash cell. When demand outstrips supply beyond a threshold, the Payment Service applies a surge multiplier to fares in that cell, which both rations scarce driver capacity (some riders wait or pay more) and incentivizes nearby drivers to reposition into the hot zone. Surge should be recalculated frequently (e.g., every 1-2 minutes) and smoothed to avoid rapid price flapping.

**2. How do you match riders and drivers efficiently in dense areas?**
Use the geospatial index (geohash or quadtree) to restrict the candidate search to a small radius first, then expand only if no driver is found. In dense areas, cap the candidate list size and rank by a cheap heuristic (ETA, distance) rather than an expensive optimization, since latency matters more than finding the theoretically optimal driver.

**3. How do you prevent double-booking a driver?**
Make "reserve driver" a single atomic, conditional operation (e.g., compare-and-set on a driver-status field, or a conditional write in the trip store) rather than an optimistic assignment. This is the first step of the trip Saga: only one concurrent request can flip a driver from "available" to "reserved," and losing requests immediately re-match against the next candidate.

**4. How do you handle payment failures mid-trip or at trip completion?**
Treat the payment charge as a saga step with a defined compensation path: on failure, retry with a backup payment method on file; if that also fails, mark the trip as completed but payment-pending and flag it for offline collection/dunning rather than trying to "undo" a trip that already physically happened. The charge call always carries an idempotency key tied to the trip ID so retries never risk a duplicate charge.

**5. How do you prevent double-charging a rider?**
Attach an idempotency key (derived deterministically from the trip ID) to every charge request sent to the Payment Service. If the same request is received twice — e.g., due to a network timeout and client retry — the Payment Service recognizes the duplicate key and returns the original charge result instead of processing a second charge.

**6. How would you design for GPS signal loss (e.g., tunnels, dense urban canyons)?**
Have the driver app buffer location pings locally and interpolate the last known trajectory (dead reckoning) so the map doesn't show the driver frozen or teleporting. On reconnect, flush buffered pings so the trip's historical trace stays accurate. The Trip Service should tolerate a location-update timeout (e.g., 30-60 seconds) before flagging a trip as "possibly stalled" rather than failing it immediately.

**7. How do you scale location updates globally?**
Shard the location store geographically using geohash prefixes as consistent-hashing keys, so each region's driver traffic lands on nearby, region-local nodes. Run regional Location Service clusters close to the drivers they serve (reducing latency and cross-region bandwidth), and treat location data as availability-favored (AP) so a regional partition doesn't take down matching elsewhere.

**8. Why use a Saga instead of two-phase commit (2PC) for the trip flow?**
2PC requires holding locks across all participating services for the duration of the transaction — impractical when a "transaction" spans a 20-minute physical car ride. A Saga breaks the flow into local transactions (reserve driver, confirm match, run trip, charge payment), each independently committed, with compensating actions defined for failure at any step — trading strict atomicity for availability and practical failure handling.

**9. What consistency model would you use for driver location data, and why?**
Eventual consistency / AP. A driver's position being a second or two stale is imperceptible to the user and doesn't cause a correctness problem, so we favor high write throughput and availability under partition (500K+ updates/sec) over strict consistency guarantees on that data.

**10. What consistency model would you use for trip state and payment data, and why?**
Strong consistency / CP. An inconsistent view of "is this driver reserved" or "was this trip charged" leads directly to double-booking or double-charging — real financial and trust costs — so we accept reduced availability (e.g., rejecting a write during a partition) in exchange for correctness on this narrow, low-volume slice of data.

**11. How would you rank multiple candidate drivers for a match beyond raw distance?**
Combine several signals into a score: estimated time to pickup (accounting for road network and traffic, not straight-line distance), driver rating, driver's current trip-completion streak/fatigue rules, and vehicle type match to rider preference. Keep the scoring function fast (sub-100ms) since it runs under the matching latency budget.

**12. How do you handle a driver going offline or losing connectivity mid-match?**
Use a heartbeat/timeout on driver location pings; if a reserved driver misses several consecutive heartbeats, the Trip Service treats the reservation as expired, releases the driver, and triggers the Saga's compensation path to re-match the rider with the next candidate — communicating the delay to the rider rather than leaving the request silently stuck.
