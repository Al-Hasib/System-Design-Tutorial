# ডায়াগ্রাম: Database Replication

## ১. Master-Slave (Leader-Follower) Replication

```mermaid
flowchart TD
    Client1[Client: Write] -->|INSERT/UPDATE/DELETE| Master[(Master / Leader)]
    Client2[Client: Read] --> LB[Load Balancer]
    LB --> Follower1[(Follower / Replica 1)]
    LB --> Follower2[(Follower / Replica 2)]
    LB --> Master
    Master -->|Async / Sync replication| Follower1
    Master -->|Async / Sync replication| Follower2
```

*ক্যাপশন: সব write একটি একক master-এ যায়, যা পরিবর্তনগুলো follower-দের কাছে স্ট্রিম করে; read master এবং যেকোনো follower জুড়ে বিতরণ করে read ক্ষমতা scale করা যায়।*

## ২. Master-Master (Multi-Leader) Replication

```mermaid
flowchart LR
    ClientUS[US Client] -->|Write| MasterUS[(Master US)]
    ClientEU[EU Client] -->|Write| MasterEU[(Master EU)]
    MasterUS <-->|Bidirectional replication| MasterEU
    MasterUS -->|Conflict?| Resolve[Conflict Resolution\nLWW / Vector Clocks]
    MasterEU -->|Conflict?| Resolve
```

*ক্যাপশন: একাধিক master প্রত্যেকে স্থানীয়ভাবে write গ্রহণ করে এবং একে অপরের কাছে replicate করে; একই রেকর্ডে একযোগে write হলে অবশ্যই conflict resolution-এর মধ্য দিয়ে যেতে হবে।*

## ৩. Master-Slave Replication-এ Failover ক্রম

```mermaid
sequenceDiagram
    participant App as Application
    participant M as Master
    participant F1 as Follower 1
    participant F2 as Follower 2

    App->>M: Write request
    M-->>F1: Replicate change
    M-->>F2: Replicate change
    Note over M: Master crashes
    App--xM: Write request fails
    Note over F1,F2: Failover detection & election
    F1->>F1: Promoted to new Master
    App->>F1: Write request (new master)
    F1-->>F2: Replicate change
```

*ক্যাপশন: master ব্যর্থ হলে, write আবার শুরু হওয়ার আগে একটি follower-কে অনুপস্থিত হিসেবে সনাক্ত করতে হয়, নির্বাচিত করতে হয়, এবং promote করতে হয় — এই ফাঁকটিই failover window।*
