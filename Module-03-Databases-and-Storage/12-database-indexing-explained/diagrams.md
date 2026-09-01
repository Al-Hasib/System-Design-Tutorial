# Diagrams: Database Indexing

## 1. B-Tree (B+Tree) Index Structure

```mermaid
flowchart TD
    Root["Root Node<br/>[30 | 60]"]

    Root --> A["Internal Node<br/>[10 | 20]"]
    Root --> B["Internal Node<br/>[40 | 50]"]
    Root --> C["Internal Node<br/>[70 | 90]"]

    A --> L1["Leaf: 5, 8<br/>&#8594 rows"]
    A --> L2["Leaf: 12, 15<br/>&#8594 rows"]
    A --> L3["Leaf: 22, 25<br/>&#8594 rows"]

    B --> L4["Leaf: 35, 38<br/>&#8594 rows"]
    B --> L5["Leaf: 42, 45<br/>&#8594 rows"]
    B --> L6["Leaf: 55, 58<br/>&#8594 rows"]

    C --> L7["Leaf: 65, 68<br/>&#8594 rows"]
    C --> L8["Leaf: 75, 80<br/>&#8594 rows"]
    C --> L9["Leaf: 95, 99<br/>&#8594 rows"]

    L1 -.->|linked| L2 -.->|linked| L3 -.->|linked| L4 -.->|linked| L5 -.->|linked| L6 -.->|linked| L7 -.->|linked| L8 -.->|linked| L9
```

*A balanced B+Tree: internal nodes act as signposts, leaf nodes hold sorted values with row pointers and are linked left-to-right, enabling both fast exact-match lookups (O(log n) descent) and efficient range scans (walking the linked leaves).*

## 2. Hash Index Structure

```mermaid
flowchart LR
    K1["Key: 'jane@example.com'"] -->|hash function| H1(("hash = 7"))
    K2["Key: 'bob@example.com'"] -->|hash function| H2(("hash = 3"))
    K3["Key: 'amy@example.com'"] -->|hash function| H3(("hash = 3"))

    H1 --> B7["Bucket 7<br/>&#8594 row pointer (jane)"]
    H2 --> B3["Bucket 3<br/>&#8594 row pointer (bob)<br/>&#8594 row pointer (amy)"]
    H3 --> B3
```

*A hash index runs each key through a hash function to jump directly to a bucket (O(1) average lookup); colliding keys (bob and amy both hash to 3) chain within the same bucket, and there is no ordering between buckets, so range queries aren't possible.*

## 3. Query Path Comparison

```mermaid
flowchart TD
    Q["Query arrives:<br/>WHERE email = 'x'"] --> Choice{Index type?}
    Choice -->|B-Tree| BT["Descend root &#8594 internal &#8594 leaf<br/>O(log n)"]
    Choice -->|Hash| HI["Compute hash &#8594 jump to bucket<br/>O(1) average"]
    Choice -->|No index| FS["Sequential scan of all rows<br/>O(n)"]

    BT --> Row["Row found via pointer"]
    HI --> Row
    FS --> Row
```

*Three paths to the same row: a B-Tree descent, a direct hash-bucket jump, or — with no index at all — a full linear scan.*
