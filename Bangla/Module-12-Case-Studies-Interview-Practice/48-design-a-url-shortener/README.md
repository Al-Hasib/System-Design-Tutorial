# Design a URL Shortener

**Difficulty:** Advanced (Capstone)
**Estimated length:** 20-30 min
**Prerequisites:**
[Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md),
[Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md),
[SQL vs NoSQL](../../Module-03-Databases-and-Storage/11-sql-vs-nosql/README.md),
[Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md),
[Caching Strategies and Cache Invalidation](../../Module-04-Caching-and-Content-Delivery/17-caching-strategies-and-cache-invalidation/README.md),
[Consistent Hashing Explained](../../Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/README.md),
[Rate Limiting Algorithms](../../Module-06-Distributed-Systems-Concepts/25-rate-limiting-algorithms/README.md)

## Learning Objectives

- Requirements clarify করা থেকে শুরু করে trade-off discussion পর্যন্ত, একটি classic system design প্রশ্নের সম্পূর্ণ mock interview end-to-end চালানো।
- Interview-এর time pressure-এর মধ্যে back-of-the-envelope capacity estimation (QPS, storage, bandwidth) অনুশীলন করা।
- আগের module থেকে আসা concept-গুলো — consistent hashing, cache-aside caching, এবং database sharding — একটি একক, নির্দিষ্ট system-এ প্রয়োগ করা।
- Counter-based এবং hash-based key generation strategy তুলনা করা এবং একটি পছন্দকে justify করা।
- একটি বাস্তব feature (custom aliases)-এর প্রেক্ষাপটে SQL এবং NoSQL storage-এর মধ্যে, এবং strong ও eventual consistency-র মধ্যে trade-off স্পষ্টভাবে ব্যাখ্যা করা।

## Script

### Hook / Intro

"Design a URL shortener" সম্ভবত industry-র সবচেয়ে common system design interview প্রশ্ন — এবং ঠিক এই কারণেই এটি এই course-এর জন্য উপযুক্ত capstone। উপর থেকে দেখতে এটি সহজ মনে হয়: একটি লম্বা URL নাও, একটি ছোট URL ফেরত দাও, এবং কেউ ক্লিক করলে redirect করো। কিন্তু এই সহজ অনুরোধের নিচে লুকিয়ে আছে এই series-এ আমরা যা কিছু কভার করেছি তার প্রায় সবকিছুই — capacity estimation, load balancing, caching, sharding, consistent hashing, এবং rate limiting। এই video-তে আমি এটা ঠিক সেভাবেই চালাব যেভাবে আমি একটি বাস্তব interview-তে চালাতাম: requirements স্পষ্ট করা, scale অনুমান করা, high-level design sketch করা, দুই-তিনটি component-এ গভীরভাবে ঢোকা, এবং তারপর trade-off নিয়ে প্রকৃত সময় ব্যয় করা। এই module-এর বাকি video-গুলো যদি দেখে থাকেন, তাহলে এখানে ব্যবহৃত প্রায় প্রতিটি অংশ চিনতে পারবেন — সেটাই মূল বিষয়।

### Step 1: Clarify Requirements

একটি box আঁকার আগেই, আমি interviewer-এর সাথে scope স্পষ্ট করি। আমি জিজ্ঞাসা করব: আমাদের প্রত্যাশিত scale কত? আমাদের কি custom, user-চয়িত alias দরকার, নাকি শুধু auto-generated short code? Link কি expire হয়? আমাদের কি click analytics দরকার? এটা কি একটি একক global service, নাকি multi-region deployment দরকার?

এই session-এর জন্য, নিচেরগুলো ঠিক করে নিই।

**Functional requirements:**
- একটি লম্বা URL দেওয়া হলে, একটি unique, ছোট alias তৈরি করা (যেমন, `short.ly/aZ9kQ2`)।
- একটি ছোট alias দেওয়া হলে, user-কে মূল লম্বা URL-এ redirect করা (HTTP 301/302)।
- User-এর বেছে নেওয়া optional custom alias সমর্থন করা।
- Link-এর জন্য optional expiration date সমর্থন করা।
- মৌলিক click analytics (count এবং timestamp) একটি nice-to-have, core নয়।

**Non-functional requirements:**
- High availability — একটি ভাঙা redirect service কখনো shared প্রতিটি link ভেঙে দেয়, তাই এখানে uptime নিখুঁত consistency-র চেয়ে বেশি গুরুত্বপূর্ণ।
- Redirect-এ low latency — read path-টা instant মনে হওয়া উচিত, আদর্শভাবে cache থেকে single-digit millisecond-এ।
- System-টি read-এ highly scalable হওয়া উচিত, কারণ link তৈরি হওয়ার চেয়ে অনেক বেশি বার share এবং click হয়।
- Short code গুলো collide করা উচিত নয়, এবং আদর্শভাবে সহজে guess/enumerate করা যাওয়া উচিত নয়।

সেই শেষ non-functional পয়েন্টটি ইতিমধ্যেই একটি গুরুত্বপূর্ণ বিষয় বলে দিচ্ছে: এটি একটি read-heavy system, তাই আমাদের বেশিরভাগ design শক্তি read (redirect) path-কে দ্রুত ও সস্তা করার দিকে যাওয়া উচিত, যেখানে write (shorten) path একটু heavier হওয়ার সামর্থ্য রাখে।

### Step 2: Capacity Estimation

চলুন এতে বাস্তব সংখ্যা বসাই, কারণ "web scale" এমন কোনো সংখ্যা নয় যা কোনো interviewer গ্রহণ করবে।

**অনুমান:**
- মাসে 500 মিলিয়ন নতুন URL লেখা হয়।
- একটি 100:1 read-to-write ratio (প্রতিটি link, গড়ে, 100 বার click হয়)।
- প্রতিটি URL record প্রায় 500 byte (লম্বা URL, short code, metadata, timestamp, owner ID)।
- Data 5 বছর ধরে retain করতে হবে।

**Traffic:**
- মাসে read = 500M × 100 = 50 বিলিয়ন redirect/মাস।
- Write QPS (গড়) = 500,000,000 / (30 × 24 × 3600 সেকেন্ড) ≈ 500,000,000 / 2,592,000 ≈ **~193 writes/sec**।
- Read QPS (গড়) = 50,000,000,000 / 2,592,000 ≈ **~19,300 reads/sec**।
- Traffic কদাচিৎ uniform হয়, তাই আমি প্রায় 2-3x গড়ের একটি peak multiplier-এর জন্য design করব — বলা যাক **peak-এ ~40,000-60,000 reads/sec**। এই peak সংখ্যাটাই আসলে আমাদের caching এবং load balancing সিদ্ধান্ত চালায়।

**Storage (5 বছর):**
- মোট record = 500M/মাস × 12 মাস × 5 বছর = **30 বিলিয়ন URL**।
- মোট storage = 30,000,000,000 × 500 byte = 15,000,000,000,000 byte = 5 বছরে raw metadata-র **~15 TB**। এটা সহজেই commodity database node-এর একটি ছোট cluster জুড়ে shard করা যায়।

**Bandwidth:**
- Write bandwidth = 193 writes/sec × 500 byte ≈ ~96 KB/sec — তুচ্ছ।
- Read bandwidth = 19,300 reads/sec × ~500 byte (redirect response) ≈ গড়ে ~9.6 MB/sec, এবং peak-এ প্রায় 20-30 MB/sec — তবুও পরিমিত, কিন্তু এটা নিশ্চিত করে যে read-ই system-এর footprint-এ প্রাধান্য পায়, storage নয়।

**Key space check:** যদি আমরা short code-গুলোকে 7 character দিয়ে base62 (`[a-zA-Z0-9]`)-এ encode করি, তাহলে আমরা পাই 62^7 ≈ 3.5 trillion সম্ভাব্য code — 5 বছরে আমাদের প্রক্ষেপিত 30 বিলিয়ন URL-এর চেয়ে আরামসে অনেক বেশি, growth-এর জন্য বিশাল headroom সহ।

এই section থেকে মূল কথা: write হালকা (প্রতি সেকেন্ডে কয়েকশ), read heavy (প্রতি সেকেন্ডে কয়েক দশ হাজার, আরও বেশি burst করতে পারে), এবং storage, absolute term-এ বড় হলেও, যথেষ্ট ছোট যে sharding সহ একটি single ভালোভাবে বেছে নেওয়া database technology এটা আরামসে সামলাতে পারবে। এই read-heavy, storage-light profile-ই এখান থেকে প্রতিটি সিদ্ধান্তকে আকার দেয়।

### Step 3: High-Level Design

সেই profile-এর ভিত্তিতে, system-এর আকৃতি এরকম:

```
Client -> Load Balancer -> App Servers (stateless) -> Cache -> Database
                                    |
                          Key Generation Service
```

- **Client** দুই ধরনের request পাঠায়: একটি লম্বা URL (এবং ঐচ্ছিকভাবে একটি custom alias/expiry) সহ `POST /shorten`, এবং redirect হওয়ার জন্য `GET /{shortCode}`।
- **Load balancer** stateless application server-এর একটি fleet-এর সামনে বসে, traffic বণ্টন করে এবং আমাদের horizontal scalability ও failover দেয় — এটা ঠিক আমরা আগে যে load balancing fundamentals কভার করেছি তা থেকেই: app server জুড়ে round-robin বা least-connections routing, health check সহ যা মৃত node সরিয়ে দেয়।
- **App server** stateless — যেকোনো server যেকোনো request handle করতে পারে — এটাই ঠিক আমাদের load balancer-এর পেছনে শুধু আরও box যোগ করে horizontal-ভাবে scale করতে দেয়।
- **Key Generation Service** write path-কে unique short code সরবরাহ করে, app server থেকে decouple করা যাতে key allocation একটি bottleneck বা collision-এর উৎস না হয়ে যায়।
- **Cache** read path-এ database-এর সামনে বসে — যেহেতু read, write-এর চেয়ে 100:1 বেশি, একটি cache বেশিরভাগ redirect traffic শুষে নেয়।
- **Database** হলো short code এবং লম্বা URL-এর মধ্যে mapping-এর, সাথে metadata (creation time, expiry, owner, click count)-এর durable source of truth।

Write path: client একটি লম্বা URL submit করে, একটি app server key generation service-কে একটি unique code চাওয়ার জন্য অনুরোধ করে (অথবা একটি চাওয়া custom alias validate করে), database-এ mapping লেখে, cache populate করে, এবং short URL ফেরত দেয়।

Read path: client `GET /{shortCode}` hit করে, app server প্রথমে cache check করে; hit হলে, সাথে সাথে redirect করে; miss হলে, database থেকে পড়ে, cache populate করে, এবং তারপর redirect করে।

### Step 4: Deep Dive on Key Components

**4a. Key generation: counter-based বনাম hash-based।** এখানে দুটি classic পদ্ধতি আছে। প্রথমটি hash-based: লম্বা URL-কে MD5 বা SHA-256-এর মধ্য দিয়ে চালাও, digest-এর প্রথম 7 character base62-encode করো, এবং সেটাকে short code হিসেবে ব্যবহার করো। এটা সহজ এবং stateless, কিন্তু collision সম্ভব, তাই database-এর বিপরীতে একটি check-and-retry loop দরকার, যা table পূর্ণ হতে থাকলে write-path latency বাড়ায়। দ্বিতীয় পদ্ধতিটি counter-based: একটি globally unique, monotonically increasing counter বজায় রাখো (একটি dedicated key-generation service-এর মতো কিছু দ্বারা backed, যা প্রতিটি app server-কে pre-allocated ID range সরবরাহ করে), এবং counter value-কে একটি short code-এ base62-encode করো। এটা কোনো collision এবং কোনো retry ছাড়াই guarantee দেয়, বিনিময়ে একটি ছোট stateful service চালাতে হয়। একটি interview-তে, আমি নির্দিষ্টভাবে counter-based পদ্ধতিটাই প্রস্তাব করব কারণ এটা hot write path থেকে collision handling সম্পূর্ণভাবে সরিয়ে দেয় — প্রতিটি app server একবারে, ধরুন, 1,000টি ID-এর একটি batch অনুরোধ করতে পারে এবং locally সেগুলো বিতরণ করতে পারে, যা key service-এর সাথে round trip-ও কমিয়ে দেয়।

**4b. Read path caching করা — এখানেই Module 4-এর cache-aside সরাসরি কাজে আসে।** যেহেতু redirect write-এর চেয়ে 100 গুণ বেশি ঘন ঘন হয়, এবং বাস্তব-জগতের link popularity একটি heavy power-law অনুসরণ করে (link-এর একটি ছোট অংশ বেশিরভাগ click-এর জন্য দায়ী), একটি cache-aside strategy স্বাভাবিক পছন্দ: app server প্রথমে cache check করে, এবং শুধুমাত্র miss হলে database-এ যায়, তারপর সে যা পড়ল তা দিয়ে cache populate করে। আমরা একটি LRU eviction policy ব্যবহার করব যাতে hot link resident থাকে এবং cold link স্বয়ংক্রিয়ভাবে বের হয়ে যায়। প্রায় ~19,300 গড় reads/sec এবং একটি heavy-tailed access pattern দেওয়া থাকলে, popularity অনুযায়ী শীর্ষ 20% link-ও cache করলে বেশিরভাগ traffic শুষে নেওয়া যাবে, যা আগে আমরা যে peak read সংখ্যা হিসাব করেছি তার চেয়ে database load-কে অনেক কম রাখবে।

**4c. Database scale করা — এখানেই Module 3 এবং 6-এর sharding এবং consistent hashing কাজে আসে।** 5 বছরে 30 বিলিয়ন row-তে, একটি single database instance এটা আরামসে ধরে রাখতে বা serve করতে পারবে না, তাই আমরা short code দিয়ে shard করি। naive পদ্ধতি — `hash(shortCode) % N` — কাজ করে যতক্ষণ না আপনি একটি shard যোগ বা বাদ দেন, সেই মুহূর্তে প্রায় প্রতিটি key remap হয় এবং আপনি একটি বিশাল, অপ্রয়োজনীয় data migration trigger করেন। এটাই ঠিক সেই সমস্যা যা consistent hashing সমাধান করে: shard (এবং সেই ক্ষেত্রে cache node-ও) একটি hash ring-এ বসে, এবং একটি node যোগ বা বাদ দিলে শুধু ring-এ তার ঠিক পাশে থাকা key-গুলো remap হয়, পুরো keyspace নয়। আমি এখানে দুটি layer-এ consistent hashing প্রয়োগ করব — আমাদের cache node জুড়ে (যেমন, একটি Redis cluster) এবং আমাদের database shard জুড়ে — যাতে traffic বাড়ার সাথে সাথে cluster-কে scale up বা down করা cache-wide বা database-wide stampede সৃষ্টি না করে।

**4d. Write path rate limit করা।** যেহেতু যে কেউ `POST /shorten` call করতে পারে, আমাদের abuse থেকে রক্ষা করা দরকার — কেউ যদি spam করতে বা আমাদের key space শেষ করতে mass URL creation script করে। এটা ঠিক Module 6-এর rate limiting সমস্যা: প্রতি API key বা প্রতি source IP-তে একটি token bucket বা sliding-window limiter, request app server-এ পৌঁছানোর আগে load balancer বা API gateway layer-এ প্রয়োগ করা।

### Step 5: Bottlenecks & Trade-offs

কোনো design সম্পূর্ণ নয় যতক্ষণ না আমরা কী ছেড়ে দিয়েছি তা বলি।

- **Custom alias বনাম auto-generated code:** custom alias UX এবং branding-এর জন্য দারুণ, কিন্তু এগুলো সেই collision সমস্যা আবার নিয়ে আসে যা counter-based generator এড়াতে design করা হয়েছিল — একটি অনুরোধ করা alias হয়তো ইতিমধ্যে নেওয়া হয়ে গেছে, যার জন্য write path-এ database-এর বিরুদ্ধে একটি uniqueness check দরকার। আমি এটাকে default auto-generated flow থেকে একটি আলাদা, একটু ধীর code path হিসেবে রাখব, একটি সহজ existence check এবং একটি reservation write সহ।
- **SQL বনাম NoSQL:** এখানে access pattern — short code দিয়ে সহজ key lookup, কোনো জটিল join বা transaction নেই — একটি NoSQL key-value বা wide-column store (যেমন DynamoDB বা Cassandra)-এর জন্য একটি textbook fit, যা একটি relational database-এর চেয়ে আরও স্বাভাবিকভাবে horizontally scale করে। তা সত্ত্বেও, একটি ভালোভাবে sharded relational database (MySQL/PostgreSQL) একটি পুরোপুরি defensible পছন্দও, বিশেষত যদি team ইতিমধ্যেই একটি চালায় এবং custom alias-এর uniqueness constraint-এর মতো জিনিসের জন্য stronger consistency guarantee চায়। এটা একটি প্রকৃত CAP theorem trade-off: availability এবং partition tolerance-এর জন্য optimize করা একটি NoSQL store আমাদের click count-এর মতো জিনিসে eventual consistency দেয়, যা analytics-এর জন্য ঠিক আছে কিন্তু alias uniqueness-এর জন্য বেশি সতর্কতা দরকার।
- **Custom alias-এ cache invalidation:** যদি পরে একজন user একটি custom alias edit বা delete করতে পারে, তাহলে পুরনো code-এর cache entry সক্রিয়ভাবে invalidate করতে হবে, শুধু expire হওয়ার জন্য ছেড়ে দিলে চলবে না — নাহলে TTL শেষ না হওয়া পর্যন্ত system stale বা deleted content-এ redirect করতেই থাকবে। এটা classic "cache invalidation দুটি কঠিন জিনিসের একটি" সমস্যা, এবং এটা স্পষ্টভাবে উল্লেখ করার মতো।
- **Scale-এ ID/key generation:** একটি single centralized counter contention এবং failure-এর একক বিন্দু। বাস্তবে, আপনি counter-টাকেই shard করবেন (যেমন, প্রতি data center-এ odd/even range, অথবা machine/region identifier embed করা Snowflake-style ID) যাতে write throughput বাড়ার সাথে সাথে কোনো একটি component bottleneck না হয়ে যায়।
- **Analytics বনাম latency:** প্রতিটি redirect-এ synchronously click counter increment করলে hot path ধীর হয়ে যাবে। সঠিক trade-off হলো click event asynchronously log করা (যেমন, একটি queue-তে) এবং সেগুলোকে একটি আলাদা analytics pipeline-এ aggregate করা, redirect-কে দ্রুত, শুধু cache-based path-এ রাখা।

### Recap

আমরা দুটি সংখ্যা দিয়ে শুরু করেছিলাম — মাসে 500M write এবং একটি 100:1 read ratio — এবং সেগুলোকেই প্রতিটি সিদ্ধান্ত চালাতে দিয়েছিলাম: একটি load balancer-এর পেছনে একটি stateless app tier, collision এড়াতে একটি counter-based key generation service, read-heavy traffic শুষে নিতে একটি cache-aside layer, এবং cache ও database উভয় জুড়ে consistent-hashed sharding যাতে কষ্টকর rebalancing ছাড়াই scale করা যায়। এই পথে আমরা custom alias, SQL বনাম NoSQL, cache invalidation, এবং ID generation নিয়ে স্পষ্ট trade-off করেছি — এই ধরনের trade-off discussion সাধারণত একটি interview-তে diagram-এর চেয়েও বেশি মূল্যবান।

### What's Next

এটা case-studies module-টা শেষ করে দেয়। যদি আপনি এই দক্ষতাগুলো আরও শাণিত করতে চান, তাহলে এই URL shortener design-টা মাথায় রেখে consistent hashing, sharding, এবং caching strategies-এর আগের deep-dive video-গুলো আবার দেখুন — আপনি লক্ষ্য করবেন যে সম্পূর্ণ ভিন্ন system জুড়ে — সেটা একটি chat app হোক, একটি news feed হোক, বা একটি payments platform হোক — একই কয়েকটি primitive কতবার পুনরায় ব্যবহৃত হয়।

## Key Takeaways

- Design করার আগে সবসময় functional এবং non-functional requirements স্পষ্ট করুন — এগুলোই নির্ধারণ করে আপনি read, write, নাকি consistency-র জন্য optimize করছেন।
- Back-of-the-envelope math (QPS, storage, bandwidth) architectural সিদ্ধান্ত চালানো উচিত, শুধু interview সাজানো নয়।
- এই system read-heavy (100:1), যে কারণে এখানে cache-aside caching এবং horizontal read scaling write optimization-এর চেয়ে বেশি গুরুত্বপূর্ণ।
- Counter-based key generation hot write path-এ collision এড়ায়; hash-based generation সহজ কিন্তু collision retry দরকার।
- Naive modulo-based sharding-এর তুলনায়, cache বা database node scale করার সময় consistent hashing data movement কমিয়ে দেয়।
- Trade-off discussion — SQL বনাম NoSQL, cache invalidation, custom alias, CAP theorem-এর প্রভাব — এগুলোতেই আসলে interview জেতা বা হারা হয়।
