# Module 6: Distributed Systems Concepts (Advanced)

এই module কোর্সের advanced core। Database, caching, এবং messaging covered হওয়ার পর, এখন আমরা সেই কঠিন সমস্যাগুলো নিয়ে কাজ করব যা তখনই দেখা দেয় যখন একটা system একটা machine-এর বদলে অনেকগুলো machine জুড়ে চলে: load কীভাবে ন্যায্যভাবে distribute করবেন, overload এবং cascading failure থেকে service-কে কীভাবে সুরক্ষা দেবেন, independent node-গুলোকে কীভাবে একটা single truth-এ agree করাবেন, এবং data একইসাথে একাধিক জায়গায় থাকলে consistency নিয়ে কীভাবে reasoning করবেন। এই concept-গুলোই "আমি একটা database ব্যবহার করতে পারি" আর "আমি একটা distributed system design করতে পারি"-এর মধ্যে পার্থক্য তৈরি করে, এবং senior-level system design interview-এ এগুলো বারবার আসে।

| # | Title | Description | Link |
|---|-------|-------------|------|
| 24 | Consistent Hashing Explained | Node-গুলোর মধ্যে key কীভাবে distribute করবেন যাতে server যোগ/বাদ দিলে যতটা সম্ভব কম data reshuffle হয় | [24-consistent-hashing-explained](./24-consistent-hashing-explained/README.md) |
| 25 | Rate Limiting Algorithms (Token Bucket, Leaky Bucket, Sliding Window) | Traffic throttle করতে এবং service সুরক্ষা দিতে ব্যবহৃত প্রধান algorithm-গুলোর তুলনা | [25-rate-limiting-algorithms](./25-rate-limiting-algorithms/README.md) |
| 26 | Circuit Breaker, Retry & Bulkhead Patterns | Resilience pattern যা একটা service-এর failure-কে পুরো system জুড়ে cascade হওয়া থেকে আটকায় | [26-circuit-breaker-retry-and-bulkhead-patterns](./26-circuit-breaker-retry-and-bulkhead-patterns/README.md) |
| 27 | Consensus Algorithms: Paxos & Raft | Failure সত্ত্বেও distributed node-গুলো কীভাবে একটা single value বা leader-এ agree করে | [27-consensus-algorithms-paxos-and-raft](./27-consensus-algorithms-paxos-and-raft/README.md) |
| 28 | Distributed Transactions: Two-Phase Commit & Saga Pattern | একাধিক service বা database জুড়ে atomic-এর মতো operation coordinate করা | [28-distributed-transactions-2pc-and-saga](./28-distributed-transactions-2pc-and-saga/README.md) |
| 29 | Data Consistency Models & Idempotency in Distributed Systems | Strong বনাম eventual বনাম causal consistency, এবং idempotency কীভাবে retry নিরাপদ রাখে | [29-data-consistency-models-and-idempotency](./29-data-consistency-models-and-idempotency/README.md) |
