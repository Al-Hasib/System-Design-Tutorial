# Module 4: Caching & Content Delivery

This module covers how systems avoid repeating expensive work by storing and reusing results at multiple layers — from in-process caches and distributed caches like Redis and Memcached, to globally distributed CDNs that serve content from locations close to the user. Caching is one of the highest-leverage tools in system design because a well-placed cache can cut latency by an order of magnitude and remove enormous load from databases and origin servers, often for a fraction of the engineering effort required to scale the underlying system itself. The trade-off — and the reason caching is also a common source of subtle bugs — is that it trades correctness/freshness guarantees for speed, so knowing the strategies, invalidation techniques, and failure modes covered here is essential for any system design interview or real production system.

## Videos in This Module

| # | Title | Description | Link |
|---|-------|-------------|------|
| 17 | Caching Strategies & Cache Invalidation (Cache-Aside, Write-Through, Write-Back) | How to decide where a cache sits in the read/write path and how to keep it from serving stale data. | [17-caching-strategies-and-cache-invalidation](./17-caching-strategies-and-cache-invalidation/README.md) |
| 18 | CDN (Content Delivery Network) Explained | How CDNs push content to edge locations near users to slash latency and offload origin servers. | [18-cdn-explained](./18-cdn-explained/README.md) |
| 19 | Distributed Caching with Redis & Memcached | Building a shared caching layer across many application servers, and choosing between Redis and Memcached. | [19-distributed-caching-redis-and-memcached](./19-distributed-caching-redis-and-memcached/README.md) |
