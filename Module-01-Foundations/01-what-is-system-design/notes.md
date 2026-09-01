# Notes: What is System Design?

## Definition

**System design** is the process of defining the architecture, components, modules, interfaces, and data flow of a software system so it satisfies a given set of requirements (scale, speed, reliability, cost).

- Focuses on **how pieces fit together**, not implementation details of a single function.
- Analogy: bricklayer (writes one brick/function well) vs. architect (decides how the whole building/system fits together).

## Why It Matters

| Reason | Detail |
|---|---|
| Interviews | Standard part of mid-to-senior software engineering interviews at most tech companies; open-ended, trade-off driven. |
| Career growth | Senior/staff engineers are expected to make architecture decisions, not just implement tickets. |
| No single right answer | Evaluated on reasoning, trade-offs, and communication — not a fixed "correct" solution. |

## Core Building Blocks (previewed, detailed later in course)

- **Client** — browser, mobile app, or another service making a request.
- **Server** — processes requests, returns responses.
- **Database** — durable, structured/unstructured data storage.
- **Cache** — fast, temporary storage layer to reduce load/latency.
- **Load balancer** — distributes incoming traffic across multiple servers.
- **Message queue** — decouples producers and consumers of data/events.

## Course Roadmap (12 Modules)

| Module | Focus |
|---|---|
| 1. Foundations | Requirements, internet basics, scalability, reliability |
| 2. Networking & Communication | HTTP, load balancing, proxies, API gateways, WebSockets |
| 3. Databases & Storage | SQL vs NoSQL, indexing, replication, sharding, CAP |
| 4. Caching & CDN | Caching strategies, CDN, distributed caches |
| 5. Messaging & Async Systems | Queues, pub/sub, event-driven, stream processing |
| 6. Distributed Systems Concepts | Consistent hashing, rate limiting, circuit breakers, consensus |
| 7. Architecture Patterns | Monolith vs microservices, service discovery, DDD |
| 8. Protocols, Formats & Security | Transport protocols (TCP/UDP/gRPC), web server internals, message formats, security fundamentals |
| 9. Database & API Internals | Transaction isolation & concurrency control, LSM trees vs B-trees, GraphQL |
| 10. Distributed Coordination & Scale Techniques | Distributed locking, logical clocks, probabilistic data structures |
| 11. Observability, Deployment & Production Operations | Logging/metrics/tracing, containers/Kubernetes, zero-downtime deploys, chaos engineering, multi-region DR |
| 12. Case Studies | URL shortener, rate limiter, chat app, news feed, file storage, video streaming, ride sharing |

## Quick Revision Bullets

- System design = architecture-level thinking, not line-by-line coding.
- Two audiences for this skill: interviewers and your future engineering team.
- Always clarify requirements before proposing a design (foreshadowing next video).
- The course builds vocabulary incrementally — later modules assume earlier ones.
