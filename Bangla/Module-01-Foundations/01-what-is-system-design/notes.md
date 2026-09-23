# Notes: System Design কী?

## Definition

**System design** হলো একটি software system-এর architecture, components, modules, interfaces, এবং data flow সংজ্ঞায়িত করার প্রক্রিয়া যাতে এটি প্রদত্ত requirements (scale, speed, reliability, cost) পূরণ করে।

- ফোকাস করে **অংশগুলো কীভাবে একসাথে খাপ খায়** তার উপর, একটি একক function-এর implementation details-এর উপর নয়।
- Analogy: bricklayer (একটি brick/function ভালোভাবে লেখে) বনাম architect (পুরো building/system কীভাবে একসাথে খাপ খায় তা ঠিক করে)।

## এটা কেন গুরুত্বপূর্ণ

| কারণ | বিস্তারিত |
|---|---|
| Interviews | বেশিরভাগ tech company-তে mid-to-senior software engineering interviews-এর একটি standard অংশ; open-ended, trade-off driven। |
| Career growth | Senior/staff engineers-দের শুধু ticket implement নয়, architecture decision নেওয়ারও প্রত্যাশা থাকে। |
| কোনো একক সঠিক উত্তর নেই | reasoning, trade-offs, এবং communication-এর উপর ভিত্তি করে evaluate করা হয় — কোনো fixed "সঠিক" সমাধানের উপর নয়। |

## Core Building Blocks (এখানে preview, course-এ পরে বিস্তারিত)

- **Client** — browser, mobile app, বা অন্য কোনো service যা একটি request করে।
- **Server** — requests process করে, responses ফেরত দেয়।
- **Database** — durable, structured/unstructured data storage।
- **Cache** — দ্রুত, সাময়িক storage layer যা load/latency কমায়।
- **Load balancer** — একাধিক server জুড়ে incoming traffic বিতরণ করে।
- **Message queue** — data/events-এর producers এবং consumers-কে decouple করে।

## Course Roadmap (12 Modules)

| Module | Focus |
|---|---|
| 1. Foundations | Requirements, internet basics, scalability, reliability |
| 2. Networking & Communication | HTTP, load balancing, proxies, API gateways, WebSockets |
| 3. Databases & Storage | SQL বনাম NoSQL, indexing, replication, sharding, CAP |
| 4. Caching & CDN | Caching strategies, CDN, distributed caches |
| 5. Messaging & Async Systems | Queues, pub/sub, event-driven, stream processing |
| 6. Distributed Systems Concepts | Consistent hashing, rate limiting, circuit breakers, consensus |
| 7. Architecture Patterns | Monolith বনাম microservices, service discovery, DDD |
| 8. Protocols, Formats & Security | Transport protocols (TCP/UDP/gRPC), web server internals, message formats, security fundamentals |
| 9. Database & API Internals | Transaction isolation ও concurrency control, LSM trees বনাম B-trees, GraphQL |
| 10. Distributed Coordination & Scale Techniques | Distributed locking, logical clocks, probabilistic data structures |
| 11. Observability, Deployment & Production Operations | Logging/metrics/tracing, containers/Kubernetes, zero-downtime deploys, chaos engineering, multi-region DR |
| 12. Case Studies | URL shortener, rate limiter, chat app, news feed, file storage, video streaming, ride sharing |

## Quick Revision Bullets

- System design = architecture-level thinking, line-by-line coding নয়।
- এই দক্ষতার জন্য দুই ধরনের audience: interviewers এবং আপনার ভবিষ্যতের engineering team।
- design প্রস্তাব করার আগে সবসময় requirements স্পষ্ট করুন (পরের video-র foreshadowing)।
- এই course ধাপে ধাপে vocabulary তৈরি করে — পরবর্তী modules আগেরগুলো ধরে নেয়।
