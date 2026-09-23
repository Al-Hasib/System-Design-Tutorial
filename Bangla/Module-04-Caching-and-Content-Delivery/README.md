# Module 4: Caching & Content Delivery

এই module-এ দেখানো হয়েছে কীভাবে system একাধিক layer-এ result store ও reuse করে ব্যয়বহুল কাজ বারবার না করার জন্য — in-process cache এবং Redis, Memcached-এর মতো distributed cache থেকে শুরু করে globally distributed CDN পর্যন্ত, যা user-এর কাছাকাছি location থেকে content serve করে। System design-এর সবচেয়ে বেশি leverage দেওয়া tool-গুলোর একটা হলো caching, কারণ একটা ভালোভাবে বসানো cache latency-কে অনেকগুণ কমিয়ে ফেলতে পারে এবং database ও origin server থেকে বিশাল load সরিয়ে নিতে পারে, প্রায়ই underlying system scale করার জন্য যে পরিমাণ engineering effort লাগে তার তুলনায় অনেক কম effort-এ। Trade-off হলো — এবং এই কারণেই caching subtle bug-এরও একটা common উৎস — এটা speed-এর বিনিময়ে correctness/freshness guarantee ছাড় দেয়, তাই এখানে covered strategy, invalidation technique, এবং failure mode জানা যেকোনো system design interview বা real production system-এর জন্য অপরিহার্য।

## এই Module-এর Video সমূহ

| # | Title | Description | Link |
|---|-------|-------------|------|
| 17 | Caching Strategies & Cache Invalidation (Cache-Aside, Write-Through, Write-Back) | Read/write path-এ cache কোথায় বসবে এবং কীভাবে এটাকে stale data serve করা থেকে আটকাবেন তা ঠিক করার উপায়। | [17-caching-strategies-and-cache-invalidation](./17-caching-strategies-and-cache-invalidation/README.md) |
| 18 | CDN (Content Delivery Network) Explained | Latency কমাতে এবং origin server-এর load কমাতে CDN কীভাবে user-এর কাছাকাছি edge location-এ content push করে। | [18-cdn-explained](./18-cdn-explained/README.md) |
| 19 | Distributed Caching with Redis & Memcached | অনেকগুলো application server জুড়ে একটা shared caching layer তৈরি করা, এবং Redis ও Memcached-এর মধ্যে বেছে নেওয়া। | [19-distributed-caching-redis-and-memcached](./19-distributed-caching-redis-and-memcached/README.md) |
