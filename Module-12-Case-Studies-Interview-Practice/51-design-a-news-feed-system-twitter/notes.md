# Cheat Sheet: Design a News Feed System (Twitter/Facebook)

Quick-reference notes for interview review. Pairs with `README.md` (full script), `diagrams.md`, `quiz.md`, and `resources.md`.

## Requirements

**Functional**
- Create posts (post store write path).
- Follow/unfollow (directed, asymmetric follow graph).
- Home timeline: posts from everyone a user follows.
- Ranked feed (recency + affinity + predicted engagement), not purely chronological.
- Out of scope: comments, likes, DMs, ads.

**Non-functional**
- Low read latency for timeline loads (reads >> writes).
- Eventual consistency is acceptable (a few seconds of delay is fine).
- Must gracefully absorb high write fan-out from celebrity posts.
- Availability favored over strong consistency (AP-leaning).

## Capacity Numbers (assumed for this session)

| Metric | Value |
|---|---|
| Daily active users (DAU) | 300 million |
| Avg posts/user/day | 2 → ~600M posts/day |
| Avg posts/sec | ~7,000 (avg), ~21,000 (peak, 3x) |
| Avg followers/user | 500 |
| Celebrity accounts (10M+ followers) | ~1 million accounts |
| Largest celebrity accounts | 50-100M followers |
| Feed reads/day | ~3 billion (10 opens/user/day) |
| Avg reads/sec | ~35,000 |
| Avg fan-out writes/sec (normal users) | ~3.5 million (7,000 posts/sec x 500 followers) |
| Single celebrity post fan-out burst | up to 50-100 million writes |
| Precomputed timeline cache size | ~24 TB (300M users x 800 entries x 100 bytes) |

## Architecture Summary

```
Client -> Load Balancer -> Post Service -> sharded Post Store (Module 3: sharding)
                                    \-> publish event -> Message Queue / Pub-Sub (Module 5, Kafka)
                                                              \-> Fan-out Service -> Follow-Graph Store
                                                                          \-> Ranking Service
                                                                          \-> Timeline Cache (Module 4: Redis, cache-aside)

Client -> Load Balancer -> Feed Service -> Timeline Cache (hit) -> hydrate posts -> rank/merge -> response
                                    \-> (cache miss / celebrity merge) -> Post Store + Follow-Graph Store
```

- Post Service: validates + persists posts, emits "new post" events.
- Fan-out Service: consumes events, decides push vs. skip per follower count.
- Ranking Service: scores candidates at fan-out time (rough) and/or read time (fine).
- Timeline Cache: Redis, sharded via consistent hashing (Module 6), sorted sets per user.
- Follow-Graph Store: optimized graph reads (who-follows-whom), sharded separately from posts.

## Key Decisions & Trade-offs

| Strategy | Write cost | Read cost | Best for | Main risk |
|---|---|---|---|---|
| Fan-out-on-write (push) | High — one write per follower per post | Low — single cache read | Normal users (~500 followers) | Celebrity posts cause massive write bursts (hot key) |
| Fan-out-on-read (pull) | Low — one write per post | High — merge posts from every followee at read time | Celebrity accounts (millions of followers) | Slow reads for users who follow many people |
| Hybrid (used in this design) | Push for normal users, skip push for celebrities | Merge in celebrity posts at read time | Production systems at Twitter/Facebook scale | Added read-path complexity (two merge sources) |

## Other Key Decisions

- **Consistent hashing** (Module 6) distributes both post-store shards and cache-cluster keys evenly, minimizes reshuffling when scaling out, and helps isolate hot keys with targeted replicas.
- **Cache-aside** (Module 4) for precomputed timelines: read cache first, rebuild from source on miss, repopulate cache.
- **Pub-sub / message queue** (Module 5, Kafka) decouples Post Service from Fan-out Service, absorbs bursty celebrity fan-out load, and enables replay/retry.
- **Eventual consistency** is an explicit, accepted trade-off — do not over-engineer for strong consistency here.
