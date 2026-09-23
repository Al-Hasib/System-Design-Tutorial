# System Design Tutorial — YouTube Playlist (বাংলা)

একটা সম্পূর্ণ System Design কোর্স, ৫৪-video-এর একটা YouTube playlist হিসেবে সাজানো, যা একজন learner-কে **একদম beginner থেকে advanced, interview-ready** engineer পর্যন্ত নিয়ে যায়। প্রতিটা video-এর নিজস্ব folder আছে, prep, record, এবং study করার জন্য যা যা দরকার তা সহ:

| File | Purpose |
|---|---|
| `README.md` | পুরো narration script — objective, prerequisite, talk track, key takeaways |
| `why.md` | কেন এই topic-টা আছে — এটা না থাকলে যে সমস্যাগুলো দেখা দেয়, এটা কীভাবে সেগুলো সমাধান করে, এবং এর বিনিময়ে কী খরচ হয় |
| `notes.md` | সংক্ষিপ্ত study notes / cheat-sheet (definition, comparison table, সংখ্যা) |
| `diagrams.md` | আলোচিত architecture বা flow visualize করা Mermaid diagram |
| `resources.md` | বাছাই করা further-reading link (official doc, paper, Wikipedia) |
| `quiz.md` | Model answer সহ Practice ও interview question |

দেখার recommended order হলো numeric (01 → 54) — পরের video-গুলো আগের video-এর concept ধরে নেয়, এবং প্রতিটা video-এর `README.md`-তে এর নির্দিষ্ট prerequisite-এর লিংক থাকে।

## Course Map

### [Module 1: Foundations](Module-01-Foundations/README.md) — *Beginner*
শুরুর জায়গা: system design আসলে কী, requirements নিয়ে কীভাবে ভাবতে হয়, client-server web কীভাবে কাজ করে, এবং দুইটা core lever (scaling, reliability) যার উপর বাকি সবকিছু গড়ে ওঠে।

| # | Video | Folder |
|---|---|---|
| 01 | What is System Design? Roadmap & How to Learn It | [01-what-is-system-design](Module-01-Foundations/01-what-is-system-design/README.md) |
| 02 | Functional vs Non-Functional Requirements | [02-functional-vs-non-functional-requirements](Module-01-Foundations/02-functional-vs-non-functional-requirements/README.md) |
| 03 | Client-Server Architecture & How the Internet Works | [03-client-server-architecture-and-how-the-internet-works](Module-01-Foundations/03-client-server-architecture-and-how-the-internet-works/README.md) |
| 04 | Scalability Basics: Vertical vs Horizontal Scaling | [04-scalability-basics-vertical-vs-horizontal-scaling](Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md) |
| 05 | Availability, Reliability, Redundancy & Fault Tolerance | [05-availability-reliability-and-fault-tolerance](Module-01-Foundations/05-availability-reliability-and-fault-tolerance/README.md) |

### [Module 2: Networking & Communication](Module-02-Networking-and-Communication/README.md) — *Beginner/Intermediate*
Client, server, এবং service-গুলো production-এ আসলে কীভাবে একে অপরের সাথে কথা বলে।

| # | Video | Folder |
|---|---|---|
| 06 | HTTP/HTTPS & REST APIs Explained | [06-http-https-and-rest-apis](Module-02-Networking-and-Communication/06-http-https-and-rest-apis/README.md) |
| 07 | Load Balancing Explained (Algorithms & L4 vs L7) | [07-load-balancing-explained](Module-02-Networking-and-Communication/07-load-balancing-explained/README.md) |
| 08 | Forward Proxy vs Reverse Proxy | [08-forward-proxy-vs-reverse-proxy](Module-02-Networking-and-Communication/08-forward-proxy-vs-reverse-proxy/README.md) |
| 09 | API Gateway & Backend-for-Frontend Pattern | [09-api-gateway-and-bff-pattern](Module-02-Networking-and-Communication/09-api-gateway-and-bff-pattern/README.md) |
| 10 | WebSockets, Long Polling & Server-Sent Events | [10-websockets-long-polling-and-sse](Module-02-Networking-and-Communication/10-websockets-long-polling-and-sse/README.md) |

### [Module 3: Databases & Storage](Module-03-Databases-and-Storage/README.md) — *Intermediate*
Data কোথায় এবং কীভাবে থাকে, replicate হয়, এবং scale করে।

| # | Video | Folder |
|---|---|---|
| 11 | SQL vs NoSQL: Choosing the Right Database | [11-sql-vs-nosql](Module-03-Databases-and-Storage/11-sql-vs-nosql/README.md) |
| 12 | Database Indexing Explained (B-Trees, Hash Indexes) | [12-database-indexing-explained](Module-03-Databases-and-Storage/12-database-indexing-explained/README.md) |
| 13 | Database Replication: Master-Slave & Master-Master | [13-database-replication](Module-03-Databases-and-Storage/13-database-replication/README.md) |
| 14 | Database Sharding & Partitioning Strategies | [14-database-sharding-and-partitioning](Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md) |
| 15 | CAP Theorem & PACELC Explained | [15-cap-theorem-and-pacelc](Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md) |
| 16 | ACID vs BASE, Normalization vs Denormalization | [16-acid-vs-base-normalization-vs-denormalization](Module-03-Databases-and-Storage/16-acid-vs-base-normalization-vs-denormalization/README.md) |

### [Module 4: Caching & Content Delivery](Module-04-Caching-and-Content-Delivery/README.md) — *Intermediate*
System design-এর সবচেয়ে বেশি leverage দেওয়া performance tool।

| # | Video | Folder |
|---|---|---|
| 17 | Caching Strategies & Cache Invalidation | [17-caching-strategies-and-cache-invalidation](Module-04-Caching-and-Content-Delivery/17-caching-strategies-and-cache-invalidation/README.md) |
| 18 | CDN (Content Delivery Network) Explained | [18-cdn-explained](Module-04-Caching-and-Content-Delivery/18-cdn-explained/README.md) |
| 19 | Distributed Caching with Redis & Memcached | [19-distributed-caching-redis-and-memcached](Module-04-Caching-and-Content-Delivery/19-distributed-caching-redis-and-memcached/README.md) |

### [Module 5: Messaging & Asynchronous Systems](Module-05-Messaging-and-Asynchronous-Systems/README.md) — *Intermediate/Advanced*
Queue, event, এবং stream দিয়ে system-কে decouple করা।

| # | Video | Folder |
|---|---|---|
| 20 | Message Queues Explained: Kafka vs RabbitMQ | [20-message-queues-kafka-vs-rabbitmq](Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/README.md) |
| 21 | Publish-Subscribe Pattern | [21-publish-subscribe-pattern](Module-05-Messaging-and-Asynchronous-Systems/21-publish-subscribe-pattern/README.md) |
| 22 | Event-Driven Architecture | [22-event-driven-architecture](Module-05-Messaging-and-Asynchronous-Systems/22-event-driven-architecture/README.md) |
| 23 | Batch Processing vs Stream Processing | [23-batch-vs-stream-processing](Module-05-Messaging-and-Asynchronous-Systems/23-batch-vs-stream-processing/README.md) |

### [Module 6: Distributed Systems Concepts](Module-06-Distributed-Systems-Concepts/README.md) — *Advanced*
কঠিন সমস্যাগুলো: consistency, consensus, এবং scale-এ failure।

| # | Video | Folder |
|---|---|---|
| 24 | Consistent Hashing Explained | [24-consistent-hashing-explained](Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/README.md) |
| 25 | Rate Limiting Algorithms (Token Bucket, Leaky Bucket, Sliding Window) | [25-rate-limiting-algorithms](Module-06-Distributed-Systems-Concepts/25-rate-limiting-algorithms/README.md) |
| 26 | Circuit Breaker, Retry & Bulkhead Patterns | [26-circuit-breaker-retry-and-bulkhead-patterns](Module-06-Distributed-Systems-Concepts/26-circuit-breaker-retry-and-bulkhead-patterns/README.md) |
| 27 | Consensus Algorithms: Paxos & Raft | [27-consensus-algorithms-paxos-and-raft](Module-06-Distributed-Systems-Concepts/27-consensus-algorithms-paxos-and-raft/README.md) |
| 28 | Distributed Transactions: Two-Phase Commit & Saga Pattern | [28-distributed-transactions-2pc-and-saga](Module-06-Distributed-Systems-Concepts/28-distributed-transactions-2pc-and-saga/README.md) |
| 29 | Data Consistency Models & Idempotency in Distributed Systems | [29-data-consistency-models-and-idempotency](Module-06-Distributed-Systems-Concepts/29-data-consistency-models-and-idempotency/README.md) |

### [Module 7: Architecture Patterns](Module-07-Architecture-Patterns/README.md) — *Advanced*
Service-কে কেন্দ্র করে পুরো system (এবং team) structure করা।

| # | Video | Folder |
|---|---|---|
| 30 | Monolith vs Microservices | [30-monolith-vs-microservices](Module-07-Architecture-Patterns/30-monolith-vs-microservices/README.md) |
| 31 | Microservices Communication & Service Discovery | [31-microservices-communication-and-service-discovery](Module-07-Architecture-Patterns/31-microservices-communication-and-service-discovery/README.md) |
| 32 | Domain-Driven Design Basics for System Design | [32-domain-driven-design-basics](Module-07-Architecture-Patterns/32-domain-driven-design-basics/README.md) |

### [Module 8: Protocols, Formats & Security](Module-08-Protocols-Formats-and-Security/README.md) — *Intermediate/Advanced*
বাকি সবকিছুর নিচের substrate: byte আসলে কীভাবে নড়াচড়া করে, একটা web server concurrent load কীভাবে handle করে, service-গুলো data format নিয়ে কীভাবে একমত হয়, এবং এই সবকিছু রক্ষা করা security fundamental।

| # | Video | Folder |
|---|---|---|
| 33 | Transport Protocols: TCP vs UDP & Where gRPC Fits | [33-transport-protocols-tcp-udp-and-grpc](Module-08-Protocols-Formats-and-Security/33-transport-protocols-tcp-udp-and-grpc/README.md) |
| 34 | Web Server Internals: Concurrency, Threading & Content Serving | [34-web-server-internals-concurrency-and-content-serving](Module-08-Protocols-Formats-and-Security/34-web-server-internals-concurrency-and-content-serving/README.md) |
| 35 | Message Formats: JSON, XML & Protocol Buffers | [35-message-formats-json-xml-and-protocol-buffers](Module-08-Protocols-Formats-and-Security/35-message-formats-json-xml-and-protocol-buffers/README.md) |
| 36 | Security Fundamentals: TLS, Encryption, AuthN/AuthZ & Firewalls | [36-security-fundamentals-tls-encryption-auth-and-firewalls](Module-08-Protocols-Formats-and-Security/36-security-fundamentals-tls-encryption-auth-and-firewalls/README.md) |

### [Module 9: Database & API Internals](Module-09-Database-and-API-Internals/README.md) — *Advanced*
Database-এ (transaction isolation, storage engine internals) আরও এক ধাপ গভীরে, এবং REST-এর একটা genuine alternative।

| # | Video | Folder |
|---|---|---|
| 37 | Transaction Isolation Levels & Concurrency Control: Locking vs. MVCC | [37-transaction-isolation-levels-and-concurrency-control](Module-09-Database-and-API-Internals/37-transaction-isolation-levels-and-concurrency-control/README.md) |
| 38 | LSM Trees vs. B-Trees: Storage Engine Internals | [38-lsm-trees-vs-b-trees-storage-engine-internals](Module-09-Database-and-API-Internals/38-lsm-trees-vs-b-trees-storage-engine-internals/README.md) |
| 39 | GraphQL: A Query-Based Alternative to REST | [39-graphql-a-query-based-alternative-to-rest](Module-09-Database-and-API-Internals/39-graphql-a-query-based-alternative-to-rest/README.md) |

### [Module 10: Distributed Coordination & Scale Techniques](Module-10-Distributed-Coordination-and-Scale-Techniques/README.md) — *Advanced*
কোনো shared clock বা unlimited memory ছাড়াই exclusive access coordinate করা, event order করা, এবং সস্তায় approximate প্রশ্নের উত্তর দেওয়া।

| # | Video | Folder |
|---|---|---|
| 40 | Distributed Locking: Redlock, ZooKeeper & etcd | [40-distributed-locking-redlock-zookeeper-and-etcd](Module-10-Distributed-Coordination-and-Scale-Techniques/40-distributed-locking-redlock-zookeeper-and-etcd/README.md) |
| 41 | Logical Clocks & Time in Distributed Systems | [41-logical-clocks-and-time-in-distributed-systems](Module-10-Distributed-Coordination-and-Scale-Techniques/41-logical-clocks-and-time-in-distributed-systems/README.md) |
| 42 | Probabilistic Data Structures: Bloom Filters, HyperLogLog & Count-Min Sketch | [42-probabilistic-data-structures-bloom-filters-hyperloglog-and-count-min-sketch](Module-10-Distributed-Coordination-and-Scale-Techniques/42-probabilistic-data-structures-bloom-filters-hyperloglog-and-count-min-sketch/README.md) |

### [Module 11: Observability, Deployment & Production Operations](Module-11-Observability-Deployment-and-Production-Operations/README.md) — *Advanced*
একটা system আসলেই live হয়ে গেলে যা যা দরকার: এটা কী করছে তা দেখা, package ও schedule করা, নিরাপদে change ship করা, failure সহ্য করে তা প্রমাণ করা, এবং একটা পুরো region down হয়ে গেলেও টিকে থাকা।

| # | Video | Folder |
|---|---|---|
| 43 | Observability: Logging, Metrics & Distributed Tracing | [43-observability-logging-metrics-and-distributed-tracing](Module-11-Observability-Deployment-and-Production-Operations/43-observability-logging-metrics-and-distributed-tracing/README.md) |
| 44 | Containers & Orchestration: Docker & Kubernetes Fundamentals | [44-containers-and-orchestration-docker-and-kubernetes-fundamentals](Module-11-Observability-Deployment-and-Production-Operations/44-containers-and-orchestration-docker-and-kubernetes-fundamentals/README.md) |
| 45 | Zero-Downtime Deployments & Database Migrations | [45-zero-downtime-deployments-and-database-migrations](Module-11-Observability-Deployment-and-Production-Operations/45-zero-downtime-deployments-and-database-migrations/README.md) |
| 46 | Testing Distributed Systems: Load Testing & Chaos Engineering | [46-testing-distributed-systems-load-testing-and-chaos-engineering](Module-11-Observability-Deployment-and-Production-Operations/46-testing-distributed-systems-load-testing-and-chaos-engineering/README.md) |
| 47 | Multi-Region Architecture & Disaster Recovery | [47-multi-region-architecture-and-disaster-recovery](Module-11-Observability-Deployment-and-Production-Operations/47-multi-region-architecture-and-disaster-recovery/README.md) |

### [Module 12: Case Studies — System Design Interview Practice](Module-12-Case-Studies-Interview-Practice/README.md) — *Advanced / Capstone*
উপরের সবকিছু real "design X" সমস্যায় apply করে পুরো mock interview: requirements → capacity estimation → high-level design → deep dive → trade-off।

| # | Video | Folder |
|---|---|---|
| 48 | Design a URL Shortener | [48-design-a-url-shortener](Module-12-Case-Studies-Interview-Practice/48-design-a-url-shortener/README.md) |
| 49 | Design a Rate Limiter (Practical System Design) | [49-design-a-rate-limiter](Module-12-Case-Studies-Interview-Practice/49-design-a-rate-limiter/README.md) |
| 50 | Design a Chat Application (like WhatsApp) | [50-design-a-chat-application-whatsapp](Module-12-Case-Studies-Interview-Practice/50-design-a-chat-application-whatsapp/README.md) |
| 51 | Design a News Feed System (like Twitter/Facebook) | [51-design-a-news-feed-system-twitter](Module-12-Case-Studies-Interview-Practice/51-design-a-news-feed-system-twitter/README.md) |
| 52 | Design a Distributed File Storage System (like Google Drive/Dropbox) | [52-design-a-distributed-file-storage-google-drive](Module-12-Case-Studies-Interview-Practice/52-design-a-distributed-file-storage-google-drive/README.md) |
| 53 | Design a Video Streaming Platform (like YouTube/Netflix) | [53-design-a-video-streaming-platform-youtube-netflix](Module-12-Case-Studies-Interview-Practice/53-design-a-video-streaming-platform-youtube-netflix/README.md) |
| 54 | Design a Ride-Sharing System (like Uber) | [54-design-a-ride-sharing-system-uber](Module-12-Case-Studies-Interview-Practice/54-design-a-ride-sharing-system-uber/README.md) |

## এই Repo কীভাবে ব্যবহার করবেন

- **একটা video record করছেন?** সেই video-এর `README.md` খুলুন — এটা একটা পুরো script, যাতে একটা intro hook, structured talking point, এবং পরের video-তে নিয়ে যাওয়ার একটা outro আছে।
- **একটা topic কেন গুরুত্বপূর্ণ তা নিশ্চিত না?** `why.md` দিয়ে শুরু করুন — এটাতে concrete সমস্যাগুলো দেখানো হয়েছে যা concept-টা না থাকলে দেখা দেয়, এটা কীভাবে প্রতিটা সমাধান করে, বিনিময়ে কী খরচ হয়, এবং কখন এটা ব্যবহার *করা উচিত না*।
- **Interview-এর জন্য পড়ছেন?** দ্রুত revision path-এর জন্য module জুড়ে `notes.md` এবং `quiz.md` দেখুন; Module 12-এর case study-গুলো সবচেয়ে ভালো শেষ rehearsal।
- **Video/slide-এর জন্য visual চান?** প্রতিটা folder-এর `diagrams.md`-তে ready-to-render Mermaid diagram আছে।
- **কোনো topic-এ আরও গভীরে যেতে চান?** `resources.md`-তে official doc-এর, এবং প্রাসঙ্গিক হলে original paper-এর লিংক আছে।
