# Further Reading & References

## Concepts

- [News Feed — Wikipedia](https://en.wikipedia.org/wiki/News_Feed) — overview of the news feed concept as popularized by Facebook.
- [Publish–subscribe pattern — Wikipedia](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) — background on the pub-sub messaging pattern used for fan-out.
- [Consistent hashing — Wikipedia](https://en.wikipedia.org/wiki/Consistent_hashing) — background on the technique used to distribute cache and shard load evenly.

## Engineering Blogs

- [Meta (Facebook) Engineering](https://engineering.fb.com/) — Meta's engineering blog, with posts on News Feed ranking, infrastructure, and large-scale systems.
- [Netflix Tech Blog](https://netflixtechblog.com/) — large-scale distributed systems, caching, and streaming architecture case studies relevant to feed-like ranking and delivery systems.
- [Google Research](https://research.google/) — publications on large-scale ranking, recommendation, and distributed systems techniques applicable to feed ranking.

## Official Documentation

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/) — official docs for the message queue / pub-sub system used for post fan-out.
- [Redis Documentation](https://redis.io/docs/latest/) — official docs for the in-memory cache used for precomputed timelines, including sorted sets and clustering (consistent hashing-based sharding).
