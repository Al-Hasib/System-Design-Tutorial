# ডায়াগ্রাম: Probabilistic Data Structures

## 1. Bloom Filter: Add এবং Check

```mermaid
flowchart TB
    subgraph Add["Adding 'apple'"]
        A1["Hash 'apple' with 3 functions"] --> A2["Set bits 2, 5, 9 to 1"]
    end

    subgraph Check1["Checking 'apple' (present)"]
        C1["Hash 'apple' - positions 2, 5, 9"] --> C2["All bits are 1 - probably present"]
    end

    subgraph Check2["Checking 'banana' (never added)"]
        D1["Hash 'banana' - positions 2, 7, 9"] --> D2["Bit 7 is 0 - definitely NOT present"]
    end
```

*একটি এলিমেন্ট যোগ করা শুধুমাত্র bit সেট করে — membership যাচাই নির্ভর করে সংশ্লিষ্ট সব bit সেট আছে কিনা তার উপর। একটি শূন্য bit নিশ্চিতভাবে অনুপস্থিতি প্রমাণ করে; সব-এক শুধুমাত্র সম্ভাব্য উপস্থিতি প্রমাণ করে।*

## 2. HyperLogLog: Leading Zero থেকে Cardinality অনুমান

```mermaid
flowchart LR
    Stream[Incoming stream of items] --> Hash[Hash each item]
    Hash --> Bucket["Route to one of many buckets<br/>based on part of the hash"]
    Bucket --> Track["Track max leading-zero count<br/>observed per bucket"]
    Track --> Combine["Combine all buckets' max values<br/>via averaging formula"]
    Combine --> Estimate["Final cardinality estimate<br/>(~1-2% error, kilobytes of memory)"]
```

*পর্যবেক্ষণ করা hash value-এ leading zero-এর দীর্ঘতর run বেশি distinct আইটেম hash করার পরিসংখ্যানগত প্রমাণ — HyperLogLog সেই পরিসংখ্যানকে একটি compact cardinality অনুমানে রূপান্তরিত করে।*

## 3. Count-Min Sketch: Frequency অনুমান

```mermaid
flowchart TB
    Item["Item: 'search-term-X'"] --> H1["Hash function 1 - row 1, column 4"]
    Item --> H2["Hash function 2 - row 2, column 9"]
    Item --> H3["Hash function 3 - row 3, column 2"]

    H1 --> V1["Counter value: 42"]
    H2 --> V2["Counter value: 57 (collision inflated this one)"]
    H3 --> V3["Counter value: 40"]

    V1 --> Min{Take minimum}
    V2 --> Min
    V3 --> Min
    Min --> Result["Estimated frequency: 40<br/>(closest to true value, since collisions only inflate)"]
```

*একই আইটেমের জন্য প্রতিটি hash function একটি ভিন্ন row-এর counter নির্দেশ করে; এদের সবার মধ্যে সর্বনিম্নটি নেওয়া অন্যান্য আইটেমের সাথে collision-এর কারণে সৃষ্ট স্ফীতি ফিল্টার করে দেয়, কারণ collision শুধুমাত্র একটি counter বাড়াতে পারে, কখনো কমাতে পারে না।*
