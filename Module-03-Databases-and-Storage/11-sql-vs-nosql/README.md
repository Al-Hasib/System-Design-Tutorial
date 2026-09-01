# SQL vs NoSQL: Choosing the Right Database

**Difficulty:** Beginner/Intermediate
**Estimated video length:** 12-18 min
**Prerequisites:** [What is System Design?](../../Module-01-Foundations/01-what-is-system-design/README.md), [Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md)

## Learning Objectives

- Explain what a relational (SQL) database is and how tables, schemas, rows, and relationships work together.
- Explain what a NoSQL database is and describe the four major categories: document, key-value, wide-column, and graph.
- Compare SQL and NoSQL across schema flexibility, horizontal scaling, consistency vs. availability, and query flexibility.
- Apply a practical decision framework to choose between SQL and NoSQL for a given system requirement.
- Recognize "polyglot persistence" — using multiple database types together in one real-world system.

## Script

### Hook / Intro

Hey everyone, welcome back. If you've made it through the foundations and scalability videos, you already know that "how do we store our data" is one of the very first questions in any system design conversation. And the very first fork in that road is almost always: SQL or NoSQL?

This question shows up in nearly every system design interview, and it trips people up not because the concepts are hard, but because most explanations either get too academic or too hand-wavy. So today we're fixing that. By the end of this video, you'll know exactly what SQL and NoSQL databases actually are, the four flavors of NoSQL you need to know, the real trade-offs between them, and a simple framework for picking the right one — with a real-world example of a company using both at once.

### What is a SQL (Relational) Database?

Let's start with SQL, because it's been the default for decades and it's the mental model most of us already have some intuition for, even if we've never thought about it explicitly.

A relational database organizes data into **tables** — think of a table like a spreadsheet. Each table has a fixed **schema**: a predefined set of columns with specific data types. A `users` table might have columns like `id` (a number), `name` (text), and `email` (text). Every **row** in that table is one record — one user — and it must fit that schema. You can't just throw in an extra field on a whim; if you want a new column, you alter the table structure for everyone.

The real power of the relational model is **relationships**. Instead of duplicating a user's information inside every order they've ever placed, you store users in a `users` table and orders in a separate `orders` table, and the orders table just holds a `user_id` — a **foreign key** that points back to the right row in `users`. When you need a full picture, you use a **JOIN** to stitch those tables together on the fly. This avoids duplication and keeps your data consistent — update the user's email once, and every order that references them automatically reflects the current data.

You talk to a relational database using **SQL** — Structured Query Language. It's declarative: you say *what* you want ("give me all orders over $100 placed in the last 30 days, joined with the customer's name") and the database figures out *how* to get it efficiently.

Finally, relational databases are known for **ACID** guarantees — Atomicity, Consistency, Isolation, Durability. In plain terms: a transaction either fully happens or fully doesn't (no half-finished bank transfers), the database always moves from one valid state to another, concurrent transactions don't step on each other's toes, and once something's committed, it survives a crash. Think Postgres, MySQL, Oracle, SQL Server — this is the world you already know.

### What is a NoSQL Database?

NoSQL — literally "not only SQL" — is really an umbrella term for databases that ditch the fixed-table-with-relationships model in favor of something more flexible, and usually something built from the ground up to scale horizontally across many machines. There isn't one NoSQL model; there are four major categories, and picking the right one matters as much as picking SQL vs. NoSQL in the first place.

**1. Document stores** — like MongoDB. Instead of rows in a table, you store whole JSON-like documents. A user and all their preferences, addresses, and recent activity might live in a single document, with no schema enforced across documents. Great when your data naturally looks like a nested object and you usually fetch it as one whole thing.

**2. Key-value stores** — like Redis or DynamoDB. The simplest model: you look up a value by a key, full stop. No queries, no joins, just blazing-fast get-and-set. Perfect for caching, session storage, and shopping carts, where you always know the exact key you're looking for.

**3. Wide-column stores** — like Cassandra (also HBase). Think of a table where each row can have a different, huge number of columns, and data is physically organized to make massive writes and range scans across a cluster extremely fast. This is the workhorse behind systems ingesting enormous volumes of time-series or event data.

**4. Graph databases** — like Neo4j. Data is stored as nodes and edges — entities and the relationships between them — optimized for traversing connections, like "friends of friends" or "products frequently bought together." When your core question is about relationships themselves rather than the entities, a graph database will run circles around a relational JOIN chain.

### Key Differences

Let's line these up side by side on the dimensions that actually matter when you're designing a system.

**Schema flexibility.** SQL enforces a rigid schema up front — good for data integrity, but every change requires a migration. NoSQL (especially document and wide-column stores) lets you add fields per-record on the fly, which is great when your data shape evolves quickly or varies between records.

**Horizontal scaling.** This is huge. Relational databases were traditionally built to scale *vertically* — bigger machine, more CPU and RAM — because JOINs and transactions across multiple machines are hard to do correctly and fast. NoSQL databases were largely built from day one to scale *horizontally* — add more commodity machines, and the database automatically spreads (shards) data across them. If you remember the vertical vs. horizontal scaling video, this is that exact trade-off showing up directly in your database choice.

**Consistency vs. availability.** Relational databases default to strong consistency — read your write immediately, everywhere. Many distributed NoSQL databases default to *eventual* consistency, trading a little bit of "read-your-write" guarantee for higher availability and lower latency during network issues. This connects to the CAP theorem, which we'll cover in depth later in this module — for now, just know: SQL leans consistency-first, many NoSQL systems let you dial toward availability-first.

**Query flexibility.** SQL's JOINs let you ask arbitrary, complex, ad-hoc questions across normalized data — fantastic when you don't know all your access patterns up front, like in analytics or reporting. NoSQL generally asks you to *denormalize* — duplicate data shaped exactly for how you'll read it — which makes the common queries extremely fast, but makes ad-hoc, unplanned queries and cross-record joins much harder or impossible.

### A Decision Framework

So how do you actually choose? Ask yourself four questions.

**One — what does your data look like?** Highly structured, relational data with lots of many-to-many relationships (orders, inventory, accounts) fits SQL naturally. Nested, self-contained objects (a user profile, a product catalog entry) fit documents. Simple lookups (a session token, a cache entry) fit key-value. Deep relationship traversal (social graphs, recommendation engines) fits graph.

**Two — what's your scale?** If you're comfortably under a few million records and a single well-tuned server, SQL will serve you fine for years — don't over-engineer. If you're expecting massive, unpredictable horizontal growth in writes or storage, a NoSQL store designed for sharding out of the box will save you pain later.

**Three — how important is strict consistency?** Money, inventory counts, anything where "read stale data" causes real damage — lean SQL and strong consistency. A social media like-count or a product view counter — eventual consistency is a perfectly acceptable trade for speed and availability.

**Four — what does your team already know, and what are your access patterns?** SQL and its query flexibility are forgiving of changing requirements — you can ask new questions of the same data. NoSQL rewards you for knowing your access patterns in advance and designing your data shape around them. If you're not sure yet how you'll query the data, that flexibility is worth a lot.

### Real-World Example

Here's the thing almost every large tech company has figured out: it's rarely "SQL or NoSQL" for the *entire* system — it's both, applied deliberately. This is called **polyglot persistence**.

Take a company like Instagram or Amazon. Amazon's order and payment systems — where a transaction must be atomic and consistent, where a double-charge or a lost order is unacceptable — run on relational databases. But Amazon's shopping cart, famously, historically ran on DynamoDB, a key-value/document store, because carts need to be blazing fast, always available even during a network partition, and don't need cross-cart JOINs. Instagram stores the core social graph — who follows whom — in a graph-optimized structure for fast traversal, while storing session data and feed caches in Redis (key-value) for speed, and historically used PostgreSQL for structured account data. Netflix uses Cassandra (wide-column) to handle the enormous, globally distributed write volume of viewing history and telemetry, while using relational stores for billing.

The lesson: don't think of this as picking one winner for your whole architecture. Think of it as picking the right tool for each *piece* of data based on its shape, its scale, and its consistency needs.

### Recap

Let's bring it together. SQL databases store structured data in tables with enforced schemas and relationships, queried with SQL, and they default to strong ACID consistency — ideal when your data is relational and integrity matters most. NoSQL is an umbrella for four categories — document, key-value, wide-column, and graph — each optimized for a different data shape and built to scale horizontally, often trading strict consistency for availability and speed. The right choice depends on your data's shape, your scale, your consistency needs, and your query patterns — and real production systems almost always mix multiple database types deliberately, rather than picking just one.

### What's Next

Now that you know *which* database model to reach for, the next question is: how do you make queries against that database fast, even as your tables grow into the millions or billions of rows? In the next video, we're diving into **Database Indexing Explained** — B-trees, hash indexes, and exactly how that "explain query plan" magic works under the hood. See you there.

## Key Takeaways

- SQL databases use fixed-schema tables, enforce relationships via foreign keys and JOINs, and provide strong ACID transaction guarantees.
- NoSQL has four major categories: document (MongoDB), key-value (Redis, DynamoDB), wide-column (Cassandra), and graph (Neo4j) — each suited to a different data shape.
- SQL favors strong consistency and flexible ad-hoc queries; NoSQL favors horizontal scalability and denormalized, access-pattern-optimized data.
- Choose based on data shape, expected scale, consistency requirements, and known query/access patterns — not by default habit.
- Real-world systems (Amazon, Instagram, Netflix) typically use polyglot persistence: multiple database types, each matched to the part of the system it fits best.
