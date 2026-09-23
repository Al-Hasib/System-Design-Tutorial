# Diagrams: Logical Clocks & Time in Distributed Systems

## ১. Lamport Timestamps: Happened-Before Ordering

```mermaid
sequenceDiagram
    participant P1 as Process 1
    participant P2 as Process 2

    Note over P1: Local event, counter = 1
    P1->>P2: Message (carries counter = 1)
    Note over P2: Local event, counter = 1
    Note over P2: Receives message, counter = max(1,1)+1 = 2
    Note over P2: Local event, counter = 3
    P2->>P1: Message (carries counter = 3)
    Note over P1: Receives message, counter = max(1,3)+1 = 4
```

*প্রতিটি local event একটি process-এর নিজস্ব counter বৃদ্ধি করে; একটি message পাওয়া counter-কে local এবং প্রাপ্ত উভয় মানের উপরে ঠেলে দেয় — নিশ্চিত করে যে causally-আগের যেকোনো event একটি ছোট সংখ্যা পাবে।*

## ২. Vector Clocks: প্রকৃত Concurrency শনাক্তকরণ

```mermaid
flowchart TB
    subgraph Causal["Case 1: Causally Related"]
        VA["Vector A = [2,0]"] -.->|"every element of A <= B"| VB["Vector B = [3,1]"]
        R1["A happened-before B"]
    end

    subgraph Concurrent["Case 2: Truly Concurrent"]
        VC["Vector C = [3,0]"]
        VD["Vector D = [0,2]"]
        R2["Neither dominates the other - C and D are concurrent (a real conflict)"]
    end
```

*যদি একটি vector প্রতিটি slot-এ অন্যটিকে dominate করে, events causally ordered। যদি কেউই dominate না করে, events স্বাধীনভাবে ঘটেছে — একটি প্রকৃত conflict, যা কোনো একক "কোনটি প্রথমে এলো" উত্তর দিয়ে সমাধান করা যায় না।*

## ৩. Vector Clocks একটি Shopping Cart Conflict সমাধান করছে

```mermaid
sequenceDiagram
    participant Phone as Phone (writes to US DC)
    participant US as US Data Center
    participant EU as EU Data Center
    participant Laptop as Laptop (writes to EU DC, stale view)

    Phone->>US: Add item X, vector [1,0]
    Laptop->>EU: Add item Y, vector [0,1] (unaware of Phone's update)
    US->>EU: Replicate vector [1,0]
    EU->>EU: Compare [1,0] vs [0,1] - neither dominates - CONCURRENT
    EU->>EU: Merge both updates instead of picking one via timestamp
```

*যেহেতু কোনো write-এর vector clock অন্যটিকে dominate করে না, system সঠিকভাবে এটিকে একটি প্রকৃত conflict হিসেবে শনাক্ত করে এবং একটি অবিশ্বাস্য wall-clock timestamp-এর ভিত্তিতে একটিকে নীরবে বাতিল করার বদলে উভয় পরিবর্তন merge করে।*
</content>
