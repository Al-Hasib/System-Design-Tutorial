# Practice & Interview Questions

**1. What is a database index, and why does it speed up queries?**
An index is a separate, organized data structure (often sorted or hashed) that maps column values to the physical location of rows, similar to the index at the back of a book. It lets the database jump directly to matching rows instead of scanning the entire table, turning an O(n) full scan into an O(log n) or O(1) lookup.

**2. What is the main cost of adding an index? Why shouldn't you index every column?**
Indexes add write amplification (every insert/update/delete must also update each index), extra storage, and ongoing maintenance (rebalancing, occasional rebuilds). Indexing every column would slow down all writes and bloat storage for benefits that may never be used, so indexes should be added only for columns actually used in filters, joins, or sorts.

**3. Describe the structure of a B+Tree index and explain why it's balanced.**
A B+Tree has a root node, internal nodes that act as signposts, and leaf nodes that hold the actual sorted values with pointers to rows; leaf nodes are linked together in sorted order. It's balanced because every path from root to leaf has the same length, guaranteeing consistent O(log n) performance regardless of which value you're searching for.

**4. Why are B-Trees good for range queries but hash indexes are not?**
B+Tree leaf nodes are stored in sorted order and linked together, so once you find the start of a range, you can walk sideways across leaves to collect all matching values. Hash indexes map values to buckets via a hash function, which intentionally scrambles order — two values that are numerically or alphabetically close can land in completely unrelated buckets, so there's no way to efficiently retrieve "everything between X and Y."

**5. What is the time complexity of a lookup in a B-Tree versus a hash index?**
A B-Tree lookup is O(log n), since you descend a fixed number of balanced tree levels. A hash index lookup is O(1) on average, since the hash function computes the bucket location directly, though heavy collisions can degrade this.

**6. Give an example of a real system that uses hash indexes or hash tables, and explain why hashing is a good fit there.**
Redis uses hash tables internally for many of its core data structures, and in-memory language constructs like Python's `dict` or Java's `HashMap` are built on hash tables. These fit because the dominant access pattern is pure key-based equality lookup with no need for range queries or sorted iteration, so the O(1) average lookup of hashing is a clear win.

**7. What is a composite (multi-column) index, and why does column order matter?**
A composite index is built across multiple columns, e.g. `(last_name, first_name)`. It efficiently serves queries that filter on a left-prefix of the columns — `last_name` alone, or `last_name AND first_name` together — but cannot efficiently serve a query filtering on `first_name` alone, because the index is physically sorted first by `last_name`.

**8. What is a covering index, and what problem does it solve?**
A covering index includes every column a query needs (in `SELECT`, `WHERE`, and `ORDER BY`), so the database can answer the query entirely from the index without a second lookup into the table itself (an "index-only scan"). It reduces I/O and latency for hot, read-heavy queries, at the cost of a larger index.

**9. In what situations should you avoid adding an index, even though it would technically speed up a query?**
On write-heavy tables (e.g., logging/event tables), where extra indexes slow down every insert; on small tables, where a full scan is already fast enough that the index overhead doesn't pay off; and on low-cardinality columns (e.g., a boolean flag), where the index can't narrow the search down much because too many rows share the same value.

**10. A table has 500 million rows, and queries filtering on `email` are slow. What would you do, and what are the trade-offs?**
First confirm with `EXPLAIN`/query plan analysis that the query is doing a full table scan due to a missing index on `email`. Add a B-Tree index on `email` (or a composite index if the query also filters/sorts on another column), since email lookups are typically exact-match or prefix-based and benefit from ordering. Trade-offs: every insert/update touching `email` now also writes to the index, storage grows, and if the table is extremely write-heavy the added index cost needs to be weighed against the read-latency win — but for a lookup-style query like this, the read speedup (potentially seconds down to milliseconds) usually far outweighs the write overhead.

**11. Why might a database choose not to use an index even when one exists on the filtered column?**
The query planner estimates cost based on statistics; if the filter matches a large fraction of the table (low selectivity), a sequential scan can actually be cheaper than jumping between the index and the table for millions of rows (this "random I/O" cost is why planners sometimes ignore indexes on low-cardinality or poorly selective columns). Stale statistics or a mismatched query pattern (e.g., filtering on a non-leading column of a composite index) can also cause the planner to skip the index.

**12. What is the difference between a clustered index and a regular (secondary) index, conceptually?**
A clustered index determines the physical storage order of the table's rows themselves (the table data lives inside the index structure, typically on the primary key) — a table can have only one. A secondary (non-clustered) index is a separate structure that stores indexed values alongside pointers back to the actual row location, and a table can have many of these to speed up different query patterns.
