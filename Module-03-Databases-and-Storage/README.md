# Module 3: Databases & Storage

Every system design interview and every real production system eventually comes down to one question: where and how do you store your data? This module covers the core decisions around choosing a database model, making reads fast with indexes, keeping data safe and available through replication, scaling writes past a single machine with sharding, and understanding the theoretical limits (CAP/PACELC) and consistency trade-offs (ACID vs BASE) that govern all of it. Get these fundamentals right and everything else you build on top — caching, messaging, microservices — has a solid foundation; get them wrong and no amount of clever architecture elsewhere will save you.

## Videos in This Module

| # | Title | Description | Link |
|---|-------|-------------|------|
| 11 | SQL vs NoSQL: Choosing the Right Database | Compares relational and non-relational databases and when to reach for each | [11-sql-vs-nosql](./11-sql-vs-nosql/README.md) |
| 12 | Database Indexing Explained (B-Trees, Hash Indexes) | How indexes make queries fast, and the internal structures that power them | [12-database-indexing-explained](./12-database-indexing-explained/README.md) |
| 13 | Database Replication: Master-Slave & Master-Master | Copying data across nodes for availability, durability, and read scaling | [13-database-replication](./13-database-replication/README.md) |
| 14 | Database Sharding & Partitioning Strategies | Splitting data across nodes to scale writes and storage horizontally | [14-database-sharding-and-partitioning](./14-database-sharding-and-partitioning/README.md) |
| 15 | CAP Theorem & PACELC Explained | The theoretical trade-offs between consistency, availability, and latency in distributed data stores | [15-cap-theorem-and-pacelc](./15-cap-theorem-and-pacelc/README.md) |
| 16 | ACID vs BASE, Normalization vs Denormalization | Transaction guarantees and data modeling trade-offs for relational and distributed systems | [16-acid-vs-base-normalization-vs-denormalization](./16-acid-vs-base-normalization-vs-denormalization/README.md) |
