# Practice & Interview Questions

1. **What is a relational (SQL) database, in your own words?**
   A database that stores data in fixed-schema tables of rows and columns, links related tables using foreign keys, and lets you query and join across them using SQL — with strong ACID transaction guarantees.

2. **What does NoSQL stand for, and why is it a misleading name?**
   "Not only SQL." It's misleading because it's not one technology or query language — it's an umbrella term covering four very different models (document, key-value, wide-column, graph) that mainly share the trait of not being traditional fixed-schema relational databases.

3. **Name the four major categories of NoSQL databases and give one example database for each.**
   Document (MongoDB), key-value (Redis or DynamoDB), wide-column (Cassandra), graph (Neo4j).

4. **What does ACID stand for, and which type of database is it typically associated with?**
   Atomicity, Consistency, Isolation, Durability — typically associated with relational (SQL) databases, guaranteeing transactions fully succeed or fail and committed data survives failures.

5. **Explain normalization vs. denormalization and which model favors each.**
   Normalization stores each fact once and links records via foreign keys/JOINs (favored by SQL, minimizes duplication). Denormalization duplicates data shaped for how it will be read, avoiding joins at read time (favored by NoSQL, trades storage/write complexity for read speed).

6. **Why do many NoSQL databases scale horizontally more easily than traditional relational databases?**
   They're typically designed without cross-node JOINs or strict multi-node transactions as a core requirement, and they partition (shard) data across nodes by design, so adding more machines increases capacity without the coordination overhead that complex relational transactions and joins would require across nodes.

7. **What is "eventual consistency," and why would a system choose it over strong consistency?**
   A guarantee that all replicas will converge to the same value eventually, but a read immediately after a write might return stale data. Systems choose it to gain higher availability and lower latency, especially during network partitions, when immediate global consistency isn't critical to correctness.

8. **Scenario: You're designing a social media app's user profile store and activity feed. Which database types would you use, and why?**
   A document store (e.g., MongoDB) fits user profiles well since each profile is a self-contained, nested object with fields that can vary per user. For the activity feed, a wide-column or key-value store (e.g., Cassandra or Redis) fits because feeds are high-volume, append-heavy, time-ordered, and read by a known access pattern (get recent activity for user X) rather than needing ad-hoc joins.

9. **Scenario: You're building the payments and order-processing backend for an e-commerce platform. Would you lean SQL or NoSQL, and why?**
   Lean SQL — payments and orders need strong consistency and atomic transactions (a payment must fully succeed or fully fail, inventory counts must stay accurate), and the data is inherently relational (users, orders, order items, products), which relational databases and ACID guarantees are built to handle safely.

10. **What is polyglot persistence, and why do large companies use it?**
    Using multiple database types within one system, each matched to the specific sub-problem it handles best — e.g., relational for financial transactions, key-value for caching/sessions, graph for social connections, wide-column for event telemetry. Companies use it because no single database model is optimal for every kind of data and access pattern in a large system.

11. **Why might a graph database outperform a relational database for a "friends of friends" query?**
    In a relational database, that query requires multiple self-joins across a relationship table, which gets expensive as the graph grows. A graph database stores relationships as first-class edges and can traverse them directly node-to-node, making multi-hop relationship queries dramatically faster.

12. **A junior engineer says "NoSQL is just the newer, better version of SQL, so we should always use it." How would you respond?**
    That's a misconception — NoSQL isn't strictly better, it's a different set of trade-offs (often less consistency and query flexibility in exchange for easier horizontal scaling and schema flexibility). For structured, relationship-heavy data needing strong consistency and complex ad-hoc queries, a relational database is often still the right — and simpler — choice.
