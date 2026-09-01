# Database Indexing Explained (B-Trees, Hash Indexes)

**Difficulty:** Intermediate | **Estimated length:** 12-18 min | **Prerequisites:** [SQL vs NoSQL: Choosing the Right Database](../11-sql-vs-nosql/README.md)

## Learning Objectives

By the end of this video, you will be able to:

- Explain what a database index is and why it makes queries dramatically faster.
- Describe the structure and behavior of a B-Tree (B+Tree) index and why it's the default choice in most relational databases.
- Describe the structure and behavior of a hash index and when it beats a B-Tree.
- Identify other common index types (composite, covering, full-text) and what problems they solve.
- Recognize situations where adding an index does more harm than good.

## Script

### Hook/Intro

Imagine you run a query on a table with 100 million rows: `SELECT * FROM users WHERE email = 'someone@example.com'`. On one database, that query comes back in 2 milliseconds. On another — same data, same hardware — it takes 40 seconds and pegs your CPU. What's the difference? In almost every case like this, the answer is one thing: an index.

Today we're going to demystify indexing. We'll cover what an index actually is, the real cost you pay for having one, and then go deep on the two workhorses of database indexing — B-Trees and hash indexes — so you understand not just "add an index here" but *why* it works and *when* to reach for which type.

### What Is an Index, Really?

At its core, a database index is a separate data structure that stores a sorted or otherwise organized copy of one or more columns, along with a pointer back to the full row on disk. Instead of scanning every single row to find what you're looking for — what databases call a "full table scan" — the database can jump almost directly to the matching rows.

The classic analogy is the index at the back of a textbook. If you want to find every mention of "mitochondria," you don't flip through all 600 pages reading each one. You go to the index, find "mitochondria," and it tells you: pages 45, 112, 380. That index is smaller than the book, it's alphabetically sorted, and it has been pre-built so lookups are fast. A database index does the same job for a column: it's a compact, ordered structure over that column's values, pointing to the location of the corresponding rows.

Same idea as a phone book, sorted by last name. If phone books were unsorted, finding someone's number would mean reading every entry. Because it's sorted, you can jump straight to the "S" section. That sorting *is* the index.

### The Cost of Indexes

Indexes aren't free, and this is the part people skip. Every index you create is extra work and extra storage:

- **Write amplification** — every `INSERT`, `UPDATE`, or `DELETE` on a table has to also update every index on that table. If you have five indexes, one write to the row can mean six total writes: one to the table (often called the heap or clustered structure) and five to keep each index in sync.
- **Storage** — an index is a full extra data structure. A handful of indexes on a large table can easily double or triple your storage footprint.
- **Index maintenance** — as data is inserted and deleted, B-Tree indexes need to rebalance, split, and occasionally get rebuilt or reorganized (fragmentation) to stay efficient. This is background work the database engine handles, but it's not zero-cost.

So indexing is fundamentally a trade-off: you're spending write performance and disk space to buy read performance. That trade-off is usually worth it for columns you filter, join, or sort on frequently — but not for every column on every table, which we'll come back to.

### B-Tree Indexes in Depth

The B-Tree — and more specifically its variant, the B+Tree — is the default index structure in PostgreSQL, MySQL's InnoDB engine, SQL Server, Oracle, and most relational databases you'll encounter.

Structurally, a B-Tree is a balanced, sorted tree. You have a root node at the top, internal nodes in the middle that act like signposts, and leaf nodes at the bottom that hold the actual indexed values. "Balanced" means every path from the root down to a leaf is the same length — there's no lopsided branch that's way deeper than the others. That balance is what guarantees consistent, predictable performance: whether you're looking up the first row or the ten-millionth, you traverse roughly the same number of levels — typically just three or four levels even for tables with millions of rows. That gives you logarithmic time complexity, O(log n), for lookups, inserts, and deletes.

In a B+Tree specifically — the variant actually used in practice — all the real data pointers live in the leaf nodes, and those leaf nodes are linked together in a chain, left to right, in sorted order. This detail matters enormously: it means once you find your starting point, you can walk sideways across the leaves to efficiently retrieve a *range* of values. That's why B-Trees are excellent for:

- Exact match lookups: `WHERE user_id = 42`
- Range queries: `WHERE created_at BETWEEN '2026-01-01' AND '2026-02-01'`
- Sorting: `ORDER BY last_name`
- Prefix matches: `WHERE last_name LIKE 'Smith%'`

Postgres and MySQL InnoDB both default to B+Tree indexes for a reason: they're the best general-purpose structure, handling equality, ranges, and ordering all in one.

### Hash Indexes in Depth

A hash index takes a completely different approach. Instead of a sorted tree, it runs the indexed value through a hash function to compute a hash code, and uses that hash code to decide which "bucket" the value's pointer goes into. To look something up, the database hashes your search value, jumps directly to the corresponding bucket, and grabs the pointer. No traversal, no comparisons down multiple levels.

That gives you O(1) — constant time — for exact-match lookups, on average, regardless of table size. For pure equality checks, `WHERE user_id = 42`, that can be even faster than a B-Tree.

But here's the catch, and it's a big one: hashing destroys order. The hash of "apple" and the hash of "apricot" have no relationship to each other, even though the words are alphabetically close. That means hash indexes **cannot** support range queries, `ORDER BY`, or prefix matching — there's no way to walk "nearby" values because nothing is stored near anything semantically. Ask a hash index for "everything between 100 and 200," and it has no better option than checking every single entry.

This is why hash indexes show up in specific, targeted places: PostgreSQL offers hash indexes as an explicit index type for equality-only workloads, Redis uses hash tables internally for many of its data structures, and in-memory hash maps (dictionaries in Python, `HashMap` in Java) are the classic general-purpose use case outside of databases entirely. But as the *default* index for a relational table, hash indexes lose to B-Trees because real applications almost always need range queries and ordering somewhere.

### Other Index Types, Briefly

A few more worth knowing:

- **Composite (multi-column) indexes** — an index built across multiple columns, like `(last_name, first_name)`. Column order matters a lot: this index efficiently serves queries filtering on `last_name` alone, or on `last_name AND first_name` together, but it can't efficiently serve a query filtering on `first_name` alone.
- **Covering indexes** — an index that includes every column a query needs, so the database can answer the query directly from the index without ever touching the underlying table. This avoids an extra lookup step and can be a huge performance win for read-heavy queries.
- **Full-text indexes** — specialized structures (often inverted indexes) built for searching words and phrases inside large text fields, used by features like product search or document search.

### When NOT to Over-Index

Given everything above, here's when adding an index is the wrong move:

- **Write-heavy tables** — if a table is dominated by inserts and updates (think logging or event tables), every extra index slows down every write. Keep indexes minimal here.
- **Small tables** — if a table only has a few hundred or a few thousand rows, a full scan is already fast; the index overhead may not pay for itself.
- **Low-cardinality columns** — a column like `is_active` (boolean) or `status` with only three possible values doesn't benefit much from indexing, because the index can't narrow the search down very far — half your rows might share the same value.

### Real-World Example

Picture a production e-commerce app. There's an `orders` table with 50 million rows, and customer support has a dashboard that runs: `SELECT * FROM orders WHERE customer_email = 'jane@example.com' ORDER BY created_at DESC`. Support agents complain the dashboard takes 8-12 seconds to load per customer lookup, and under load it's spiking database CPU.

Running `EXPLAIN` on that query shows a sequential scan — the database is reading all 50 million rows and filtering in memory because there's no index on `customer_email`. The fix: `CREATE INDEX idx_orders_customer_email ON orders(customer_email, created_at DESC);` — a composite index that covers both the filter and the sort order.

After adding it, the same query drops from 8-12 seconds to under 10 milliseconds, because the database now jumps straight to the matching rows via the B-Tree, already in the right sorted order for `created_at DESC`, instead of scanning the entire table. The trade-off: every new order insert now also updates this index, and the index itself consumes extra disk space — but for a table that's read far more often than it's written (support dashboards, order history pages), that's a trade worth making every time.

### Recap

Let's tie it together. An index is a separate, organized structure over your data that turns slow linear scans into fast targeted lookups — like the index in the back of a book. That speed isn't free: you pay in write overhead, storage, and maintenance. B-Trees are the balanced, sorted, general-purpose workhorse, giving you O(log n) performance for equality, ranges, and ordering — which is why they're the default in Postgres and MySQL. Hash indexes trade that flexibility for raw speed, giving O(1) exact-match lookups but nothing for ranges or sorting. And beyond those two, composite indexes, covering indexes, and full-text indexes solve more specialized problems. The skill isn't "index everything" — it's knowing exactly which columns earn an index and which don't.

### What's Next

Indexing solves the problem of finding data fast on a single database. But what happens when one database server isn't enough — when you need your data to survive a hardware failure, or you need to scale out reads across multiple machines? That's exactly what we're covering in the next video: **Database Replication: Master-Slave & Master-Master**. See you there.

## Key Takeaways

- An index is a separate, ordered structure that maps column values to row locations, avoiding full table scans.
- Indexes trade write performance and storage for read performance — every index adds overhead to every insert, update, and delete.
- B-Tree (B+Tree) indexes are balanced, sorted trees offering O(log n) performance; they're the default in Postgres and MySQL InnoDB because they handle equality, range queries, and sorting.
- Hash indexes offer O(1) average-case exact-match lookups but cannot support range queries or ordering, since hashing destroys value order.
- Composite indexes speed up multi-column filters (column order matters), covering indexes let queries be answered from the index alone, and full-text indexes serve text search.
- Avoid over-indexing write-heavy tables, small tables, and low-cardinality columns — the overhead may outweigh the benefit.
