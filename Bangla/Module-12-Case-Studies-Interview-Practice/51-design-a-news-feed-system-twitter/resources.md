# আরো পড়ুন ও রেফারেন্স

## Concepts

- [News Feed — Wikipedia](https://en.wikipedia.org/wiki/News_Feed) — Facebook দ্বারা জনপ্রিয় হওয়া news feed ধারণার একটি সংক্ষিপ্ত বিবরণ।
- [Publish–subscribe pattern — Wikipedia](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) — fan-out-এর জন্য ব্যবহৃত pub-sub messaging প্যাটার্নের পটভূমি।
- [Consistent hashing — Wikipedia](https://en.wikipedia.org/wiki/Consistent_hashing) — cache ও shard load সমানভাবে বিতরণ করার জন্য ব্যবহৃত কৌশলের পটভূমি।

## Engineering Blog

- [Meta (Facebook) Engineering](https://engineering.fb.com/) — Meta-এর engineering blog, যেখানে News Feed ranking, infrastructure, এবং বৃহৎ-স্কেলের সিস্টেম নিয়ে post রয়েছে।
- [Netflix Tech Blog](https://netflixtechblog.com/) — বৃহৎ-স্কেলের distributed systems, caching, এবং streaming architecture কেস স্টাডি, যা feed-এর মতো ranking ও delivery সিস্টেমের সাথে প্রাসঙ্গিক।
- [Google Research](https://research.google/) — বৃহৎ-স্কেলের ranking, recommendation, এবং distributed systems কৌশলের উপর প্রকাশনা, যা feed ranking-এ প্রযোজ্য।

## Official Documentation

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/) — post fan-out-এর জন্য ব্যবহৃত message queue / pub-sub সিস্টেমের অফিসিয়াল ডকুমেন্টেশন।
- [Redis Documentation](https://redis.io/docs/latest/) — precomputed timeline-এর জন্য ব্যবহৃত in-memory cache-এর অফিসিয়াল ডকুমেন্টেশন, sorted sets ও clustering (consistent hashing-ভিত্তিক sharding) সহ।
