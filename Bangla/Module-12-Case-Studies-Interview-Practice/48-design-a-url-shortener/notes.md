# Interview Cheat-Sheet: URL Shortener

`README.md`-এর দ্রুত-রেফারেন্স সহায়ক। Interview-এর আগে দ্রুত ঝালিয়ে নিতে এটা ব্যবহার করুন।

## Requirements

**Functional**
- একটি লম্বা URL-কে একটি unique short alias-এ shorten করা।
- একটি short alias-কে তার মূল লম্বা URL-এ redirect করা (HTTP 301/302)।
- Optional custom (user-চয়িত) alias সমর্থন করা।
- Optional link expiration সমর্থন করা।
- (Nice-to-have) মৌলিক click analytics: count + timestamp।

**Non-functional**
- High availability (একটি ভাঙা redirect প্রতিটি shared link ভেঙে দেয়)।
- Low-latency redirect (cache থেকে single-digit ms)।
- Highly scalable read (read-heavy workload)।
- Short code unique এবং enumerate/guess করা কঠিন হতে হবে।

## Capacity Estimation

| Metric | অনুমান / হিসাব | ফলাফল |
|---|---|---|
| নতুন URL লেখা হয়েছে | মাসে 500M | 500,000,000 |
| Read:write ratio | 100:1 | — |
| Redirect (read) | 500M × 100 | মাসে 50B |
| Write QPS (গড়) | 500M / 2,592,000 সেকেন্ড | ~193 writes/sec |
| Read QPS (গড়) | 50B / 2,592,000 সেকেন্ড | ~19,300 reads/sec |
| Read QPS (peak, গড়ের 2-3x) | ~19,300 × 2-3 | ~40,000-60,000 reads/sec |
| মোট record (5 বছর) | 500M × 12 × 5 | 30B record |
| Storage (5 বছর) | 30B × 500 byte/record | ~15 TB |
| Write bandwidth | 193/sec × 500 byte | ~96 KB/sec |
| Read bandwidth (গড়) | 19,300/sec × 500 byte | ~9.6 MB/sec |
| Key space (base62, 7 character) | 62^7 | ~3.5 trillion code |

**উপসংহার:** Read-heavy (100:1), storage মোট হিসেবে বড় কিন্তু sharding দিয়ে সামলানো যায়, write হালকা। Design অগ্রাধিকার: fast cached read > write throughput।

## High-Level Architecture

```
Client -> Load Balancer -> App Servers (stateless) -> Cache -> Database
                                  |
                        Key Generation Service
```

- **Load balancer**: stateless app server জুড়ে traffic বণ্টন করে; health check মৃত node সরিয়ে দেয়।
- **App servers**: stateless, horizontally scalable, shorten (write) এবং redirect (read) — দুই ধরনের request-ই handle করে।
- **Key Generation Service**: unique short code সরবরাহ করে (counter-based, প্রতি server-এ pre-allocated ID range)।
- **Cache**: DB-এর সামনে একটি cache-aside layer (যেমন, Redis); heavy-tailed link popularity-র কারণে বেশিরভাগ read traffic শুষে নেয়; LRU eviction।
- **Database**: short-code -> long-URL mapping + metadata-এর durable store; consistent hashing ব্যবহার করে sharded।

**Write path:** client -> app server -> key gen service (code নাও) -> DB-তে write করো -> cache populate করো -> short URL ফেরত দাও।

**Read path:** client -> app server -> cache lookup -> (hit: সাথে সাথে redirect) / (miss: DB পড়ো -> cache populate করো -> redirect করো)।

## মূল Design সিদ্ধান্ত ও Trade-off

| সিদ্ধান্ত | Option A | Option B | সুপারিশ / Trade-off |
|---|---|---|---|
| Key generation | Hash-based (MD5/SHA + base62) | Counter-based (pre-allocated ID range) | Counter-based hot write path-এ collision retry এড়ায় |
| Storage | SQL (MySQL/PostgreSQL, sharded) | NoSQL (DynamoDB/Cassandra) | NoSQL সরল key-value access pattern-এ ফিট করে ও horizontally scale করে; strong uniqueness constraint গুরুত্বপূর্ণ হলে SQL ঠিক আছে |
| Consistency | Strong consistency | Eventual consistency (CAP: AP পছন্দ করো) | Click count-এর জন্য eventual consistency গ্রহণযোগ্য; custom alias uniqueness-এর জন্য stronger guarantee দরকার |
| Sharding strategy | `hash(key) % N` | Consistent hashing | Node যোগ/বাদ দিলে consistent hashing remapped key কমিয়ে দেয় |
| Custom alias | সবসময় অনুমতি দাও | শুধু auto-generate | Custom alias-এর জন্য uniqueness check দরকার (অতিরিক্ত write-path খরচ) |
| Cache invalidation | শুধু TTL | Update/delete-এ সক্রিয় invalidation | Alias edit/delete করা গেলে stale redirect এড়াতে সক্রিয় invalidation দরকার |
| Click analytics | Synchronous increment | Async event logging (queue + pipeline) | Async redirect path দ্রুত রাখে; analytics আলাদাভাবে aggregate হয় |
| Rate limiting | কোনোটাই নয় | প্রতি API key বা IP-তে token bucket / sliding window | Abuse এবং key-space exhaustion প্রতিরোধ করতে write path-এ (`POST /shorten`) দরকার |

## ধারাবাহিকভাবে ব্যবহৃত Terminology

Sharding, consistent hashing, load balancer, cache-aside, CAP theorem, base62 encoding, LRU eviction, stateless app servers, rate limiting (token bucket / sliding window)।
