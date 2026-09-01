# Diagrams: Design a News Feed System (Twitter/Facebook)

## 1. Overall Architecture

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

*Caption: Writes flow from the Post Service through a sharded post store and a Kafka-based pub-sub layer into the Fan-out and Ranking services, which populate a consistently-hashed Redis timeline cache; reads are served by the Feed Service primarily from that cache.*

## 2. Hybrid Fan-out Sequence: Celebrity Post vs. Normal Post

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

*Caption: Normal-user posts are pushed to followers' caches at write time, while celebrity posts skip the expensive push and are merged into each follower's feed at read time instead, avoiding a hot-key write burst.*
