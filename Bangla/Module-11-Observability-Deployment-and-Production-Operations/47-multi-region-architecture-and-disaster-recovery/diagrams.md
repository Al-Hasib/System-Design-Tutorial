# Diagrams: Multi-Region Architecture & Disaster Recovery

## ১. Active-Passive Failover

```mermaid
flowchart TB
    subgraph Before["Before Disaster"]
        U1[Users] --> A1["Region A (Active)<br/>Serving all traffic"]
        A1 -.->|Replicate data| P1["Region B (Passive)<br/>Standby, not serving traffic"]
    end

    subgraph After["After Region A Fails"]
        U2[Users] --> F["Failover:<br/>DNS/routing updated"]
        F --> P2["Region B (now Active)<br/>Serving all traffic"]
    end
```

*স্বাভাবিক অবস্থায়, Region B replicated data নিয়ে অলস অবস্থায় থাকে। একটি failure-এর পর, traffic তার দিকে redirect করা হয় — কিন্তু সেই redirection নিজেই সময় নেয়, যার জন্য RTO একটি গ্রহণযোগ্য সীমা নির্ধারণ করে।*

## ২. Active-Active: প্রতিটি Region Traffic পরিবেশন করে

```mermaid
flowchart TB
    U1[Users near Region A] --> A["Region A<br/>Serving live traffic"]
    U2[Users near Region B] --> B["Region B<br/>Serving live traffic"]
    A <-->|Bidirectional replication| B

    Note["If Region A fails,<br/>Region B simply absorbs its traffic<br/>- no failover delay"]
```

*উভয় region একই সাথে traffic পরিবেশন করে, নৈকট্য অনুযায়ী route করা হয় — একটি fail করলে, "take over" করার জন্য কোনো একক point নেই, কিন্তু উভয় জায়গায় গৃহীত write অবশ্যই মিলিয়ে নিতে (reconcile) হবে।*

## ৩. একটি Timeline-এ RTO এবং RPO

```mermaid
flowchart LR
    LastBackup["Last successful<br/>replicated write"] -->|"RPO window<br/>(max acceptable data loss)"| Disaster["Disaster occurs"]
    Disaster -->|"RTO window<br/>(max acceptable downtime)"| Recovered["System back online,<br/>serving traffic again"]
```

*RPO disaster থেকে পেছনের দিকে পরিমাপ করে — কতটা সাম্প্রতিক data হারানো যেতে পারত। RTO disaster থেকে সামনের দিকে পরিমাপ করে — service পুনরুদ্ধার হতে কতক্ষণ লাগে। দুটোই ব্যবসা-নির্ধারিত লক্ষ্য যা প্রকৃত architecture পছন্দ চালিত করে।*
