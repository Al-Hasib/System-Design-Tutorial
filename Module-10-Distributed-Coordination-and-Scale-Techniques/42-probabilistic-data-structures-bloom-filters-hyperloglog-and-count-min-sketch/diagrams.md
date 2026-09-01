# Diagrams: Probabilistic Data Structures

## 1. Bloom Filter: Add and Check

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
*Adding an element only ever sets bits — checking membership relies on all relevant bits being set. One zero bit proves absence with certainty; all-ones only proves probable presence.*

## 2. HyperLogLog: Estimating Cardinality from Leading Zeros

```mermaid
flowchart LR
    Stream[Incoming stream of items] --> Hash[Hash each item]
    Hash --> Bucket["Route to one of many buckets<br/>based on part of the hash"]
    Bucket --> Track["Track max leading-zero count<br/>observed per bucket"]
    Track --> Combine["Combine all buckets' max values<br/>via averaging formula"]
    Combine --> Estimate["Final cardinality estimate<br/>(~1-2% error, kilobytes of memory)"]
```
*Longer runs of leading zeros in observed hash values are statistical evidence of having hashed more distinct items — HyperLogLog turns that statistic into a compact cardinality estimate.*

## 3. Count-Min Sketch: Estimating Frequency

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
*Each hash function points to a different row's counter for the same item; taking the minimum across all of them filters out the inflation caused by collisions with other items, since collisions can only ever increase a counter, never decrease it.*
