# System Design Tutorial — YouTube Playlist

A complete System Design course, structured as a 39-video YouTube playlist that takes a learner from **absolute beginner to advanced, interview-ready** engineer. Every video has its own folder with everything needed to prep, record, and study from:

| File | Purpose |
|---|---|
| `README.md` | Full narration script — objectives, prerequisites, the talk track, key takeaways |
| `notes.md` | Condensed study notes / cheat-sheet (definitions, comparison tables, numbers) |
| `diagrams.md` | Mermaid diagrams visualizing the architecture or flow discussed |
| `resources.md` | Curated further-reading links (official docs, papers, Wikipedia) |
| `quiz.md` | Practice & interview questions with model answers |

Recommended viewing order is numeric (01 → 39) — later videos assume concepts from earlier ones, and each video's `README.md` links back to its specific prerequisites.

## Course Map

### [Module 1: Foundations](Module-01-Foundations/README.md) — *Beginner*
The entry point: what system design even is, how to think about requirements, how the client-server web works, and the two core levers (scaling, reliability) everything else builds on.

| # | Video | Folder |
|---|---|---|
| 01 | What is System Design? Roadmap & How to Learn It | [01-what-is-system-design](Module-01-Foundations/01-what-is-system-design/README.md) |
| 02 | Functional vs Non-Functional Requirements | [02-functional-vs-non-functional-requirements](Module-01-Foundations/02-functional-vs-non-functional-requirements/README.md) |
| 03 | Client-Server Architecture & How the Internet Works | [03-client-server-architecture-and-how-the-internet-works](Module-01-Foundations/03-client-server-architecture-and-how-the-internet-works/README.md) |
| 04 | Scalability Basics: Vertical vs Horizontal Scaling | [04-scalability-basics-vertical-vs-horizontal-scaling](Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md) |
| 05 | Availability, Reliability, Redundancy & Fault Tolerance | [05-availability-reliability-and-fault-tolerance](Module-01-Foundations/05-availability-reliability-and-fault-tolerance/README.md) |

### [Module 2: Networking & Communication](Module-02-Networking-and-Communication/README.md) — *Beginner/Intermediate*
How clients, servers, and services actually talk to each other in production.

| # | Video | Folder |
|---|---|---|
| 06 | HTTP/HTTPS & REST APIs Explained | [06-http-https-and-rest-apis](Module-02-Networking-and-Communication/06-http-https-and-rest-apis/README.md) |
| 07 | Load Balancing Explained (Algorithms & L4 vs L7) | [07-load-balancing-explained](Module-02-Networking-and-Communication/07-load-balancing-explained/README.md) |
| 08 | Forward Proxy vs Reverse Proxy | [08-forward-proxy-vs-reverse-proxy](Module-02-Networking-and-Communication/08-forward-proxy-vs-reverse-proxy/README.md) |
| 09 | API Gateway & Backend-for-Frontend Pattern | [09-api-gateway-and-bff-pattern](Module-02-Networking-and-Communication/09-api-gateway-and-bff-pattern/README.md) |
| 10 | WebSockets, Long Polling & Server-Sent Events | [10-websockets-long-polling-and-sse](Module-02-Networking-and-Communication/10-websockets-long-polling-and-sse/README.md) |

### [Module 3: Databases & Storage](Module-03-Databases-and-Storage/README.md) — *Intermediate*
Where and how data lives, replicates, and scales.

| # | Video | Folder |
|---|---|---|
| 11 | SQL vs NoSQL: Choosing the Right Database | [11-sql-vs-nosql](Module-03-Databases-and-Storage/11-sql-vs-nosql/README.md) |
| 12 | Database Indexing Explained (B-Trees, Hash Indexes) | [12-database-indexing-explained](Module-03-Databases-and-Storage/12-database-indexing-explained/README.md) |
| 13 | Database Replication: Master-Slave & Master-Master | [13-database-replication](Module-03-Databases-and-Storage/13-database-replication/README.md) |
| 14 | Database Sharding & Partitioning Strategies | [14-database-sharding-and-partitioning](Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md) |
| 15 | CAP Theorem & PACELC Explained | [15-cap-theorem-and-pacelc](Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md) |
| 16 | ACID vs BASE, Normalization vs Denormalization | [16-acid-vs-base-normalization-vs-denormalization](Module-03-Databases-and-Storage/16-acid-vs-base-normalization-vs-denormalization/README.md) |

### [Module 4: Caching & Content Delivery](Module-04-Caching-and-Content-Delivery/README.md) — *Intermediate*
The highest-leverage performance tools in system design.

| # | Video | Folder |
|---|---|---|
| 17 | Caching Strategies & Cache Invalidation | [17-caching-strategies-and-cache-invalidation](Module-04-Caching-and-Content-Delivery/17-caching-strategies-and-cache-invalidation/README.md) |
| 18 | CDN (Content Delivery Network) Explained | [18-cdn-explained](Module-04-Caching-and-Content-Delivery/18-cdn-explained/README.md) |
| 19 | Distributed Caching with Redis & Memcached | [19-distributed-caching-redis-and-memcached](Module-04-Caching-and-Content-Delivery/19-distributed-caching-redis-and-memcached/README.md) |

### [Module 5: Messaging & Asynchronous Systems](Module-05-Messaging-and-Asynchronous-Systems/README.md) — *Intermediate/Advanced*
Decoupling systems with queues, events, and streams.

| # | Video | Folder |
|---|---|---|
| 20 | Message Queues Explained: Kafka vs RabbitMQ | [20-message-queues-kafka-vs-rabbitmq](Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/README.md) |
| 21 | Publish-Subscribe Pattern | [21-publish-subscribe-pattern](Module-05-Messaging-and-Asynchronous-Systems/21-publish-subscribe-pattern/README.md) |
| 22 | Event-Driven Architecture | [22-event-driven-architecture](Module-05-Messaging-and-Asynchronous-Systems/22-event-driven-architecture/README.md) |
| 23 | Batch Processing vs Stream Processing | [23-batch-vs-stream-processing](Module-05-Messaging-and-Asynchronous-Systems/23-batch-vs-stream-processing/README.md) |

### [Module 6: Distributed Systems Concepts](Module-06-Distributed-Systems-Concepts/README.md) — *Advanced*
The hard problems: consistency, consensus, and failure at scale.

| # | Video | Folder |
|---|---|---|
| 24 | Consistent Hashing Explained | [24-consistent-hashing-explained](Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/README.md) |
| 25 | Rate Limiting Algorithms (Token Bucket, Leaky Bucket, Sliding Window) | [25-rate-limiting-algorithms](Module-06-Distributed-Systems-Concepts/25-rate-limiting-algorithms/README.md) |
| 26 | Circuit Breaker, Retry & Bulkhead Patterns | [26-circuit-breaker-retry-and-bulkhead-patterns](Module-06-Distributed-Systems-Concepts/26-circuit-breaker-retry-and-bulkhead-patterns/README.md) |
| 27 | Consensus Algorithms: Paxos & Raft | [27-consensus-algorithms-paxos-and-raft](Module-06-Distributed-Systems-Concepts/27-consensus-algorithms-paxos-and-raft/README.md) |
| 28 | Distributed Transactions: Two-Phase Commit & Saga Pattern | [28-distributed-transactions-2pc-and-saga](Module-06-Distributed-Systems-Concepts/28-distributed-transactions-2pc-and-saga/README.md) |
| 29 | Data Consistency Models & Idempotency in Distributed Systems | [29-data-consistency-models-and-idempotency](Module-06-Distributed-Systems-Concepts/29-data-consistency-models-and-idempotency/README.md) |

### [Module 7: Architecture Patterns](Module-07-Architecture-Patterns/README.md) — *Advanced*
Structuring whole systems (and teams) around services.

| # | Video | Folder |
|---|---|---|
| 30 | Monolith vs Microservices | [30-monolith-vs-microservices](Module-07-Architecture-Patterns/30-monolith-vs-microservices/README.md) |
| 31 | Microservices Communication & Service Discovery | [31-microservices-communication-and-service-discovery](Module-07-Architecture-Patterns/31-microservices-communication-and-service-discovery/README.md) |
| 32 | Domain-Driven Design Basics for System Design | [32-domain-driven-design-basics](Module-07-Architecture-Patterns/32-domain-driven-design-basics/README.md) |

### [Module 8: Case Studies — System Design Interview Practice](Module-08-Case-Studies-Interview-Practice/README.md) — *Advanced / Capstone*
Full mock interviews applying everything above to real "design X" problems: requirements → capacity estimation → high-level design → deep dive → trade-offs.

| # | Video | Folder |
|---|---|---|
| 33 | Design a URL Shortener | [33-design-a-url-shortener](Module-08-Case-Studies-Interview-Practice/33-design-a-url-shortener/README.md) |
| 34 | Design a Rate Limiter (Practical System Design) | [34-design-a-rate-limiter](Module-08-Case-Studies-Interview-Practice/34-design-a-rate-limiter/README.md) |
| 35 | Design a Chat Application (like WhatsApp) | [35-design-a-chat-application-whatsapp](Module-08-Case-Studies-Interview-Practice/35-design-a-chat-application-whatsapp/README.md) |
| 36 | Design a News Feed System (like Twitter/Facebook) | [36-design-a-news-feed-system-twitter](Module-08-Case-Studies-Interview-Practice/36-design-a-news-feed-system-twitter/README.md) |
| 37 | Design a Distributed File Storage System (like Google Drive/Dropbox) | [37-design-a-distributed-file-storage-google-drive](Module-08-Case-Studies-Interview-Practice/37-design-a-distributed-file-storage-google-drive/README.md) |
| 38 | Design a Video Streaming Platform (like YouTube/Netflix) | [38-design-a-video-streaming-platform-youtube-netflix](Module-08-Case-Studies-Interview-Practice/38-design-a-video-streaming-platform-youtube-netflix/README.md) |
| 39 | Design a Ride-Sharing System (like Uber) | [39-design-a-ride-sharing-system-uber](Module-08-Case-Studies-Interview-Practice/39-design-a-ride-sharing-system-uber/README.md) |

## How to Use This Repo

- **Recording a video?** Open that video's `README.md` — it's a full script with an intro hook, structured talking points, and an outro leading into the next video.
- **Studying for an interview?** Skim `notes.md` and `quiz.md` across modules for a fast-revision path; Module 8's case studies are the best final rehearsal.
- **Want visuals for the video/slides?** `diagrams.md` in each folder has ready-to-render Mermaid diagrams.
- **Going deeper on a topic?** `resources.md` links to official docs and, where relevant, the original papers.
