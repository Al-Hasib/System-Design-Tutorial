# Study Notes: Database Indexing

## Definitions

- **Index** — a separate, organized data structure built over one or more columns of a table that maps column values to the physical location of the corresponding rows, allowing the database to avoid scanning the entire table.
- **Full table scan (sequential scan)** — reading every row in a table to find matches; what an index lets you avoid.
- **B-Tree / B+Tree** — a balanced, sorted tree structure. In a B+Tree, all data pointers live in linked, sorted leaf nodes, enabling efficient range scans. The default index type in most relational databases (PostgreSQL, MySQL InnoDB, SQL Server, Oracle).
- **Hash index** — an index that maps a value to a bucket via a hash function, giving average O(1) exact-match lookups but no ordering.
- **Cardinality** — the number of distinct values in a column relative to the number of rows. High cardinality (e.g., email, user ID) benefits more from indexing than low cardinality (e.g., boolean flags).
- **Composite (multi-column) index** — an index built over more than one column, in a specific column order.
- **Covering index** — an index that contains all the columns a query needs, so the query can be satisfied entirely from the index without a separate lookup into the table (sometimes called an "index-only scan").
- **Write amplification** — the extra write work caused by having to update every index whenever the underlying row changes.

## B-Tree vs Hash Index

| Aspect | B-Tree (B+Tree) | Hash Index |
|---|---|---|
| Lookup type | Exact match and range | Exact match only |
| Range queries (`<`, `>`, `BETWEEN`) | Yes — efficient, via linked sorted leaves | No — must scan all entries |
| Ordering / `ORDER BY` support | Yes — data is stored sorted | No — hashing destroys order |
| Prefix match (`LIKE 'abc%'`) | Yes | No |
| Time complexity (lookup) | O(log n) | O(1) average |
| Insert/update/delete cost | O(log n), plus occasional node splits/rebalancing | O(1) average, occasional bucket resizing/rehashing |
| Storage overhead | Moderate | Generally lower per entry, but hash collisions add overhead |
| Typical use cases | Primary keys, foreign keys, sorting, range filters, general-purpose queries | Pure equality lookups (cache keys, exact ID lookups) |
| Examples | PostgreSQL default index, MySQL InnoDB default index, SQL Server clustered/nonclustered indexes | PostgreSQL `USING hash` index, Redis internal hash tables, in-memory hash maps (Python dict, Java `HashMap`) |

## Composite Indexes

- An index over multiple columns, e.g. `(last_name, first_name)`.
- Column **order matters**: the index efficiently supports queries filtering on a **left-prefix** of the columns:
  - `WHERE last_name = 'Smith'` — efficient (uses the index).
  - `WHERE last_name = 'Smith' AND first_name = 'Jane'` — efficient (uses the full index).
  - `WHERE first_name = 'Jane'` alone — cannot efficiently use this index (not a left-prefix).
- Useful for queries that filter on one column and sort on another, e.g. `(customer_email, created_at DESC)`.

## Covering Indexes

- Includes every column referenced by a query (in the `SELECT`, `WHERE`, and `ORDER BY` clauses).
- Lets the database perform an "index-only scan" — it never has to fetch the full row from the table heap.
- Great for read-heavy, latency-sensitive queries, at the cost of a larger index (more columns stored).

## Key Numbers / Complexity

- B-Tree lookup, insert, delete: **O(log n)** — for a table with millions of rows, this is typically only 3-4 tree levels to traverse.
- Hash index lookup: **O(1)** average case (can degrade with heavy hash collisions).
- Full table scan (no index): **O(n)** — must inspect every row.
- More indexes = more write cost: each index adds roughly one additional write per `INSERT`/`UPDATE`/`DELETE` on the indexed column(s).

## When to Use Each — Summary

- **Use a B-Tree** (the safe default) when you need: range queries, sorting, prefix matching, or a mix of query patterns on the same column. This covers the vast majority of application queries.
- **Use a hash index** when: the access pattern is purely equality lookups (never ranges, never sorting), and you want the fastest possible exact-match performance — common in caching layers and in-memory key-value lookups rather than as a general relational index.
- **Use a composite index** when queries consistently filter/sort on the same combination of columns together; order columns from most-selective/most-commonly-filtered to least.
- **Use a covering index** for hot, latency-critical read queries where avoiding the extra table lookup matters.
- **Avoid indexing** write-heavy tables aggressively, small tables, and low-cardinality columns — the overhead often isn't worth the marginal read speedup.
