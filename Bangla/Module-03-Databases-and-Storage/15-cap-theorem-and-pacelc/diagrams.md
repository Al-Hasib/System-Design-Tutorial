# ডায়াগ্রাম: CAP Theorem ও PACELC

Mermaid-এর কোনো native triangle/Venn shape নেই, তাই নিচের CAP সম্পর্কটি তিনটি বৈশিষ্ট্যের একটি graph হিসেবে আনুমানিকভাবে দেখানো হয়েছে, যেখানে "দুটি বেছে নাও" জুটিগুলো চিহ্নিত করা আছে।

## ১. CAP Theorem — তিনটির মধ্যে দুটি বেছে নাও

```mermaid
graph TD
    C["Consistency (C)"]
    A["Availability (A)"]
    P["Partition Tolerance (P)"]

    C ---|"CP: consistent, may reject requests during a partition"| P
    A ---|"AP: available, may return stale data during a partition"| P
    C -.->|"CA only possible without a network partition — not realistic for multi-node systems"| A

    style C fill:#4C6EF5,color:#fff
    style A fill:#12B886,color:#fff
    style P fill:#F59F00,color:#fff
```

*Caption: একটি distributed system একই সময়ে Consistency, Availability, এবং Partition Tolerance-এর মধ্যে শুধু দুটির সম্পূর্ণ guarantee দিতে পারে — যেহেতু partition অনিবার্য, বাস্তব সিস্টেমগুলোকে CP বা AP বেছে নিতে হয়।*

## ২. একটি Network Partition-এর সময় Read/Write — CP বনাম AP

```mermaid
sequenceDiagram
    participant Client
    participant NodeA as Node A (reachable)
    participant NodeB as Node B (unreachable, other side of partition)

    Note over NodeA,NodeB: Network partition occurs between Node A and Node B

    rect rgb(240, 230, 200)
    Note over Client,NodeB: CP System Behavior
    Client->>NodeA: Write/Read request
    NodeA->>NodeB: Attempt to confirm quorum/majority
    NodeB-->>NodeA: No response (partitioned)
    NodeA-->>Client: Error / request rejected (cannot confirm consistency)
    end

    rect rgb(210, 235, 225)
    Note over Client,NodeB: AP System Behavior
    Client->>NodeA: Write/Read request
    NodeA-->>Client: Response served locally (may be stale vs. Node B)
    Note over NodeA,NodeB: Reconciliation happens later once partition heals
    end
```

*Caption: একটি CP system-এ, Node A request প্রত্যাখ্যান করে যখন এটি partition-এর অপরপাশে থাকা Node B-র সাথে সমঝোতা নিশ্চিত করতে পারে না; একটি AP system-এ, Node A সাথে সাথে জবাব দেয় এবং partition সেরে যাওয়ার পর যেকোনো বিচ্যুতি মিলিয়ে নেয়।*

## ৩. PACELC সিদ্ধান্ত প্রবাহ

```mermaid
flowchart TD
    Start["Is the system currently experiencing a network Partition?"]
    Start -->|Yes| PChoice["Choose: Availability (A) or Consistency (C)"]
    Start -->|No, Else: normal operation| EChoice["Choose: Latency (L) or Consistency (C)"]

    PChoice --> PA["PA: keep serving requests\n(e.g., Cassandra, DynamoDB)"]
    PChoice --> PC["PC: reject/block requests\n(e.g., MongoDB majority, ZooKeeper)"]

    EChoice --> EL["EL: respond fast, weaker consistency\n(e.g., Cassandra, DynamoDB default)"]
    EChoice --> EC["EC: wait for agreement, stronger consistency\n(e.g., MongoDB majority, ZooKeeper)"]
```

*Caption: PACELC একটি দ্বিতীয় সিদ্ধান্ত যোগ করে যা partition ছাড়াও প্রযোজ্য — স্বাভাবিক কার্যক্রমের সময় Latency-কে Consistency-র বিনিময়ে দেওয়া।*
