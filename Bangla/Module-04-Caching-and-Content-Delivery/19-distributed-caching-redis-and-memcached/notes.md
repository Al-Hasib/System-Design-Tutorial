# স্টাডি নোটস: Distributed Caching with Redis & Memcached

## সংজ্ঞাসমূহ

- **Local (in-process) cache**: একটি cache যা একটি একক application instance-এর মেমরিতে সংরক্ষিত থাকে; দ্রুত, কিন্তু একাধিক instance জুড়ে অসামঞ্জস্যপূর্ণ এবং পুনরাবৃত্ত।
- **Distributed cache**: একটি shared caching layer, application process-এর বাহিরে, যা সব application instance নেটওয়ার্কের মাধ্যমে access করে, cached data-এর একটি একক সামঞ্জস্যপূর্ণ দৃশ্য প্রদান করে।
- **Sharding**: একটি dataset-কে একাধিক cache node জুড়ে বিভক্ত করা, সাধারণত প্রতিটি key hash করে এর নির্ধারিত node ঠিক করা হয়।
- **Consistent hashing**: একটি hashing scheme যা cluster-এ node যোগ/বাদ দেওয়ার সময় key redistribution কমিয়ে আনে (Module 6-এ বিস্তারিত আলোচিত)।
- **RDB (Redis Database file)**: Redis-এর point-in-time snapshot persistence পদ্ধতি।
- **AOF (Append-Only File)**: Redis-এর write-ahead-log-শৈলীর persistence পদ্ধতি যা প্রতিটি write operation log করে।
- **Pub/Sub**: একটি messaging pattern যেখানে publisher-রা channel-এ message পাঠায় এবং subscriber-রা সেগুলো গ্রহণ করে; Redis-এ স্বাভাবিকভাবেই সমর্থিত।

## Redis বনাম Memcached

| বৈশিষ্ট্য | Redis | Memcached |
|---|---|---|
| Data model | সমৃদ্ধ: strings, lists, sets, sorted sets, hashes, streams, geospatial | সাধারণ key-value (শুধু byte strings) |
| Persistence | হ্যাঁ — RDB snapshots এবং/অথবা AOF log | না — restart-এ data হারিয়ে যায় |
| Replication | হ্যাঁ — built-in primary/replica replication | কোনো native replication নেই |
| Clustering / Sharding | Redis Cluster (node জুড়ে 16,384 hash slots) | Client-side sharding (সাধারণত client-এ implement করা consistent hashing) |
| Threading model | Command execution-এর জন্য মূলত single-threaded event loop (নতুন version-এ কিছু I/O threading) | Multi-threaded |
| অতিরিক্ত ফিচার | Pub/Sub, transactions (MULTI/EXEC), Lua scripting, complex type-এ atomic ops | সাধারণ get/set/incr ছাড়া কিছু নেই |
| Eviction policies | একাধিক configurable policy (LRU, LFU, random, TTL-based, ইত্যাদি) | LRU |
| সাধারণ use cases | Caching + session store + leaderboards + rate limiting + pub/sub + lightweight queues | সাধারণ, high-throughput key-value caching |
| পরিপক্বতা/জটিলতা | বেশি ফিচার, বেশি configuration surface | ন্যূনতম, চালানো সহজ |

## Local Cache বনাম Distributed Cache

| দিক | Local (In-Process) Cache | Distributed Cache |
|---|---|---|
| গতি | সবচেয়ে দ্রুত (কোনো network hop নেই) | Local-এর চেয়ে ধীর, তবুও DB-এর চেয়ে অনেক দ্রুত |
| Server জুড়ে consistency | অসামঞ্জস্যপূর্ণ — প্রতিটি instance-এর নিজস্ব কপি | সামঞ্জস্যপূর্ণ — একক shared দৃশ্য |
| Memory efficiency | Instance প্রতি পুনরাবৃত্ত | একবার সংরক্ষিত, shared |
| App server-এর সাথে scale করে? | Instance যোগ করলে cache কার্যকরভাবে "reset" হয়ে যায় | Cache app server সংখ্যা থেকে স্বাধীনভাবে scale করে |

## Sharding / Data Distribution

| পদ্ধতি | বর্ণনা | দুর্বলতা |
|---|---|---|
| Modulo hashing (hash(key) % N) | সরল, deterministic | Node যোগ/বাদ দিলে প্রায় সব key নতুন করে সাজে |
| Consistent hashing | Key এবং node একটি hash ring-এ mapped; node পরিবর্তিত হলে শুধু একটি অংশ key স্থানান্তরিত হয় | সঠিকভাবে implement করা বেশি জটিল |
| Redis Cluster hash slots | Node জুড়ে বিতরণ করা নির্দিষ্ট 16,384 slot; node পরিবর্তিত হলে slot পুনর্নির্ধারিত হয় (পুরো rehash নয়) | Redis-নির্দিষ্ট পদ্ধতি |

## মূল সংখ্যা / সাধারণ নিয়ম

- Distributed cache access সাধারণত একটি local in-process cache-এর তুলনায় sub-millisecond থেকে low-single-digit-millisecond network latency যোগ করে, কিন্তু সাধারণ database query-র চেয়ে অনেক দ্রুতই থাকে (প্রায়ই 10-100x দ্রুত)।
- Redis Cluster ঠিক 16,384টি hash slot ব্যবহার করে, cluster-এর আকার নির্বিশেষে।
- Memcached-এর multi-threaded architecture সাধারণ get/set workload-এ, অনেক CPU core-সম্পন্ন machine-এ, প্রতি node-এ বেশি raw throughput দিতে পারে।

## সারসংক্ষেপ

- একটি application server-এর বেশি scale করলে local cache কাজ করে না — সেগুলো inconsistency এবং duplicated memory তৈরি করে।
- একটি distributed cache একটি shared, network-accessible caching layer যা সব application instance ব্যবহার করে।
- Memcached: সহজ, multi-threaded, pure key-value, কোনো persistence/replication নেই — সরল high-throughput caching-এর জন্য চমৎকার।
- Redis: সমৃদ্ধ data structures, persistence, replication, clustering, pub/sub — আরও বহুমুখী, সাধারণভাবে ব্যবহৃত default পছন্দ।
- Cache cluster জুড়ে data ছড়ানো হয় sharding-এর মাধ্যমে, আদর্শভাবে consistent hashing ব্যবহার করে যাতে cluster scale করার সময় বিঘ্ন কমানো যায়।
