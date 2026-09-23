# অধ্যয়ন নোট: Caching Strategies & Cache Invalidation

## সংজ্ঞাসমূহ

- **Cache**: একটি দ্রুতগতির, সাধারণত memory-backed storage layer যেটি ডেটার copy ধরে রাখে যাতে ভবিষ্যতের request-গুলো ব্যয়বহুল গণনা বা I/O পুনরাবৃত্তি না করেই সার্ভ করা যায়।
- **Cache hit**: এমন একটি request যা সরাসরি cache থেকে সার্ভ করা হয়।
- **Cache miss**: এমন একটি request যা cache-এ পাওয়া যায় না এবং source of truth-এর দিকে (database, API, computation) ফিরে যেতে হয়।
- **Cache hit ratio**: hits / (hits + misses)। সাধারণত বেশি ভালো, কিন্তু absolute ট্রাফিক volume-এর সাপেক্ষে মূল্যায়ন করতে হবে।
- **TTL (Time To Live)**: স্বয়ংক্রিয়ভাবে expire হওয়ার আগে একটি cache entry বৈধ বলে বিবেচিত হওয়ার সময়কাল।
- **Stale data**: Cached data যা আর source of truth-এর বর্তমান অবস্থার সাথে মেলে না।
- **Cache stampede / thundering herd**: একই সময়ে অনেক concurrent request cache miss করা (যেমন, একটি hot key expire হওয়ার পরে) এবং origin-কে অপ্রতিরোধ্য করে ফেলা।

## Caching Strategies তুলনা

| স্ট্র্যাটেজি | কে পরিচালনা করে | Write path | Read path | Consistency | সবচেয়ে উপযুক্ত |
|---|---|---|---|---|---|
| Cache-Aside (Lazy Loading) | Application | App DB-তে লেখে; cache আলাদাভাবে আপডেট/invalidate করা হয় (অথবা মোটেই হয় না) | App প্রথমে cache চেক করে, miss হলে DB পড়ে তারপর cache পূরণ করে | Eventually consistent (TTL/invalidation পর্যন্ত) | সাধারণ-উদ্দেশ্য, read-heavy workload |
| Read-Through | Cache library/service | Cache-aside-এর মতো অথবা write-through-এর সাথে জোড়া লাগানো | App সবসময় cache-কে জিজ্ঞেস করে; cache miss হলে ভেতরে ভেতরে DB থেকে লোড করে | Cache-aside-এর মতোই | Cache library সমর্থন করলে app কোড সরল করা |
| Write-Through | Cache + application (synchronous) | Write একসাথে cache এবং DB-তে, synchronously যায় | Cache-এ সবসময় তাজা ডেটা থাকে | Strong (একটি সম্পন্ন write-এর পরে cache কখনো stale হয় না) | Read-heavy ডেটা যেখানে correctness গুরুত্বপূর্ণ, মাঝারি write volume |
| Write-Back (Write-Behind) | Cache (asynchronous) | Write শুধু cache-এ যায়; cache পরে, async-ভাবে DB-তে flush করে | দ্রুত, cache-এ তাজা | Cache তাজা; DB পিছিয়ে থাকতে পারে; cache crash হলে data loss-এর ঝুঁকি | Write-heavy workload যেখানে কম write latency দরকার |
| Write-Around | সরাসরি Database | Write সরাসরি DB-তে যায়, cache বাইপাস করে | Write-এর পরে প্রথম read একটি cache miss (cache-aside এটা পূরণ করে) | কদাচিৎ পড়া হওয়া writeগুলোর কারণে cache দূষণ এড়ায় | Write-once/read-rarely ডেটা (যেমন, log, audit trail) |

## Cache Invalidation পদ্ধতি

| পদ্ধতি | এটা কীভাবে কাজ করে | সুবিধা | অসুবিধা |
|---|---|---|---|
| TTL / Expiration | N সেকেন্ড পরে entry স্বয়ংক্রিয়ভাবে expire হয় | সহজ, self-healing, কোনো অতিরিক্ত কোড পথ নেই | Staleness window; সঠিক TTL বেছে নেওয়া একটি ভারসাম্যের কাজ |
| Explicit Invalidation | আন্ডারলাইং ডেটা পরিবর্তন হলেই app cache entry মুছে/আপডেট করে | চাহিদা অনুযায়ী freshness | প্রতিটি mutation পথে প্রয়োগ করতে হবে; একটা মিস করা সহজ এবং তা বাগ ঘটায় |
| Write-Through (invalidation হিসেবে) | প্রতিটি write-এর সাথে synchronously cache আপডেট হয় | কোনো staleness নেই | অতিরিক্ত write latency |

## Eviction Policies

| পলিসি | কী evict করে | নোট |
|---|---|---|
| LRU (Least Recently Used) | সবচেয়ে দীর্ঘ সময় ধরে অ্যাক্সেস না হওয়া entry | সবচেয়ে সাধারণ default; ভালো general-purpose heuristic |
| LFU (Least Frequently Used) | সবচেয়ে কম মোট বার অ্যাক্সেস হওয়া entry | যখন জনপ্রিয়তা recency-চালিত না হয়ে সময়ের সাথে স্থিতিশীল থাকে, তখন ভালো |
| FIFO (First In, First Out) | ব্যবহার নির্বিশেষে সবচেয়ে পুরনো insert করা entry | সবচেয়ে সহজ, প্রায়শই LRU/LFU-এর চেয়ে কম কার্যকর |

## গুরুত্বপূর্ণ সংখ্যা / সাধারণ নিয়ম

- RAM অ্যাক্সেস latency: ~১০০ ন্যানোসেকেন্ড; SSD: ~১০০ মাইক্রোসেকেন্ড; HDD: ~১০ মিলিসেকেন্ড — RAM ডিস্কের চেয়ে ~১,০০,০০০ গুণ দ্রুত হতে পারে।
- একটি ভালোভাবে tune করা cache-aside সিস্টেম প্রায়ই read-heavy workload-এর জন্য cache hit ratio ৯০-৯৫%-এর ওপরে ঠেলে দিতে পারে।
- এমনকি ৯৫% hit ratio-তেও, ৫% request এখনো origin-কে আঘাত করে — উচ্চ স্কেলে (যেমন, ১০,০০০ req/s) সেটা এখনো database-এ পৌঁছানো ৫০০ req/s।
- সাধারণ web-page/API TTL কয়েক সেকেন্ড (hot, ঘন ঘন পরিবর্তনশীল ডেটা) থেকে ঘণ্টা (বেশিরভাগ static ডেটা) পর্যন্ত হয়।

## সারসংক্ষেপ

- Caching উচ্চ-প্রভাবশালী: তুলনামূলকভাবে কম implementation খরচে বড় latency ও load জয়।
- আপনার read/write অনুপাত এবং staleness বনাম write latency বনাম data-loss ঝুঁকি সহ্য করার ক্ষমতার ওপর ভিত্তি করে একটি স্ট্র্যাটেজি বাছুন।
- সবসময় একটি caching স্ট্র্যাটেজির সাথে একটি স্পষ্ট invalidation এবং eviction পরিকল্পনা জোড়া লাগান — একটি invalidate না-করা cache একটি বাগ-উৎপাদক।
- Lock, request coalescing, অথবা jittered TTL ব্যবহার করে hot key-তে cache stampede থেকে রক্ষা পান।
