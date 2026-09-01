# Module 9: Database & API Internals

Module 3 taught you *when* to reach for a relational database versus a NoSQL store, and Module 8 covered how services agree on wire formats. This module goes one level deeper on both fronts. On the database side, we open up what "ACID" and "index" actually mean at the storage-engine level: how a database enforces isolation between concurrent transactions, and how the underlying data structure — B-tree or LSM-tree — shapes whether a database is optimized for reads or writes. On the API side, we look at GraphQL as a genuine alternative to REST, not just a buzzword, and understand exactly what problem it solves that REST structurally can't. None of this is required to pass a beginner system design interview, but it's exactly the kind of depth that separates "I know the vocabulary" from "I understand why the vocabulary exists."

## Videos in This Module

| # | Title | Description | Link |
|---|-------|-------------|------|
| 37 | Transaction Isolation Levels & Concurrency Control: Locking vs. MVCC | What actually happens when two transactions touch the same row at the same time — isolation levels, locking, and how MVCC avoids readers blocking writers. | [37-transaction-isolation-levels-and-concurrency-control](37-transaction-isolation-levels-and-concurrency-control/README.md) |
| 38 | LSM Trees vs. B-Trees: Storage Engine Internals | The data structure underneath your database's index, and why write-heavy NoSQL stores reach for a fundamentally different one than traditional relational databases. | [38-lsm-trees-vs-b-trees-storage-engine-internals](38-lsm-trees-vs-b-trees-storage-engine-internals/README.md) |
| 39 | GraphQL: A Query-Based Alternative to REST | The over-fetching/under-fetching problem REST can't structurally solve, and how GraphQL's single, client-driven query endpoint addresses it — along with its own real trade-offs. | [39-graphql-a-query-based-alternative-to-rest](39-graphql-a-query-based-alternative-to-rest/README.md) |
