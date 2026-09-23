# ডায়াগ্রাম: একটি নিউজ ফিড সিস্টেম ডিজাইন করা (Twitter/Facebook)

## ১. সামগ্রিক Architecture

```mermaid
flowchart LR
    Client[Client App]
    LB[Load Balancer]
    PostSvc[Post Service]
    FeedSvc[Feed Service]
    Queue[(Message Queue / Pub-Sub<br/>Kafka)]
    FanoutSvc[Fan-out Service]
    RankSvc[Ranking Service]
    PostDB[(Sharded Post Store)]
    FollowDB[(Follow-Graph Store)]
    Cache[(Timeline Cache<br/>Redis, consistent hashing)]

    Client --> LB
    LB --> PostSvc
    LB --> FeedSvc

    PostSvc --> PostDB
    PostSvc --> Queue
    Queue --> FanoutSvc
    FanoutSvc --> FollowDB
    FanoutSvc --> RankSvc
    RankSvc --> Cache

    FeedSvc --> Cache
    FeedSvc --> PostDB
    FeedSvc --> FollowDB
    Cache --> FeedSvc
    FeedSvc --> Client
```

*Caption: Writes প্রবাহিত হয় Post Service থেকে একটি sharded post store এবং একটি Kafka-ভিত্তিক pub-sub layer দিয়ে Fan-out ও Ranking service-এ, যা একটি consistently-hashed Redis timeline cache পূরণ করে; reads প্রধানত সেই cache থেকে Feed Service দ্বারা সার্ভ করা হয়।*

## ২. Hybrid Fan-out Sequence: Celebrity Post বনাম Normal Post

```mermaid
sequenceDiagram
    participant U as User (Author)
    participant PS as Post Service
    participant Q as Message Queue (Kafka)
    participant FO as Fan-out Service
    participant FG as Follow-Graph Store
    participant C as Timeline Cache (Redis)
    participant F as Follower (Feed Service)

    U->>PS: Create post
    PS->>PS: Persist to sharded Post Store
    PS->>Q: Publish "new post" event

    Q->>FO: Consume event

    alt Normal user (~500 followers)
        FO->>FG: Fetch follower list
        FO->>C: Push post ID into each follower's<br/>precomputed timeline (fan-out-on-write)
        F->>C: Read timeline (cache hit, includes new post)
    else Celebrity account (10M+ followers)
        FO->>FO: Detect follower count above threshold
        FO->>Q: Skip write-time fan-out (avoid hot-key burst)
        F->>C: Read own precomputed timeline (cache hit)
        F->>FG: Fetch small list of followed celebrities
        F->>PS: Fetch celebrity's recent posts (fan-out-on-read)
        F->>F: Merge + rank celebrity posts with cached timeline
    end
```

*Caption: সাধারণ ব্যবহারকারীর post write সময়েই followers-এর cache-এ push করা হয়, অন্যদিকে celebrity post ব্যয়বহুল push এড়িয়ে গিয়ে read সময়ে প্রতিটি follower-এর feed-এ merge করা হয়, যা একটি hot-key write burst এড়িয়ে চলে।*
