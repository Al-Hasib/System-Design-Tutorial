# কেন এই বিষয়টি গুরুত্বপূর্ণ: Distributed Caching with Redis & Memcached

> **এক বাক্যে:** যে মুহূর্তে আপনি একটির বেশি application server চালান, একটি in-process cache আর cache থাকে না, বরং N-সংখ্যক অসামঞ্জস্যপূর্ণ cache হয়ে যায় — একটি shared cache tier সেটাই ঠিক করে, এবং দেখা যায় এটি আরও অর্ধ ডজন সমস্যাও সমাধান করে।

## এই ধারণার আগের পৃথিবী

আপনি local process memory-তে cache করেন। এটি দ্রুত, বিনামূল্যের, এবং চমৎকারভাবে কাজ করে — একটি server-এ।

দ্বিতীয় server যোগ করলেই সমস্যা শুরু হয়। প্রতিটির নিজস্ব কপি থাকে, তাই একজন ব্যবহারকারী একটি request-এ একটি value দেখতে পারে এবং পরের request-এ একটি ভিন্ন value দেখতে পারে, এলোমেলোভাবে। Hit rate ধসে পড়ে: দশটি server থাকলে, একটি cached item শুধুমাত্র 1/10 ভাগ traffic-এর জন্য কার্যকর, তাই আপনি দশটি কপি warm করার জন্য দশগুণ কাজ করছেন। যখন কোনো value পরিবর্তিত হয়, তখন আপনাকে প্রতিটি node-এ এটি invalidate করতে হয় — এবং এটি করার কোনো নির্ভরযোগ্য পদ্ধতি নেই। Deploy-এর সময় সমস্ত cache একসাথে মুছে যায়, যা সবচেয়ে খারাপ মুহূর্তে database-এর উপর পুরো traffic ফেলে দেয়।

এদিকে কিছু জিনিস আসলে local memory-তে থাকতেই পারে না: একটি session অবশ্যই load balancer যে server বেছে নেয় সেখানেও দৃশ্যমান হতে হবে, একটি rate-limit counter অবশ্যই global হতে হবে যাতে এর অর্থ থাকে, এবং একটি lock যদি per-process হয় তাহলে তা মূল্যহীন।

## এটি যে সমস্যাগুলো সমাধান করে

### ১. একটি fleet জুড়ে cache inconsistency
**আপনি যা দেখেন:** ব্যবহারকারীরা refresh করলে পুরনো এবং নতুন value-এর মধ্যে পাল্টাতে দেখেন। Invalidation "কাজ করে" কিন্তু শুধু মাঝে মাঝে।

**কেন এটি ঘটে:** N-সংখ্যক স্বাধীন cache, যাদের মধ্যে কোনো সমন্বয় নেই।

**একটি shared cache কীভাবে এটি সমাধান করে:** একটি logical cache, প্রতিটি key-এর একটি value, প্রতিটি server-এর কাছে দৃশ্যমান। একবার invalidate করুন এবং এটি সবার জন্যই invalidate হয়ে যায়। Local caching থেকে সরে যাওয়ার এটাই মূল কারণ।

### ২. State যা সঠিক হওয়ার জন্য অবশ্যই shared হতে হবে
**আপনি যা দেখেন:** ব্যবহারকারীর request বিভিন্ন server-এ পৌঁছালে session ভেঙে যায়। Rate limit উদ্দিষ্ট traffic-এর 10 গুণ অনুমতি দেয় কারণ প্রতিটি server স্বাধীনভাবে গণনা করে। একটি scheduled job একই সময়ে সবগুলো বারোটি node-এ চলে।

**কেন এটি ঘটে:** এই তিনটিরই *globally* দৃশ্যমান state প্রয়োজন। Per-process state per-process উত্তর দেয়।

**Redis কীভাবে এটি সমাধান করে:** Atomic operation সহ একটি দ্রুত shared store। Counter-এর জন্য `INCR`, lock-এর জন্য expiry সহ `SETNX`, session-এর জন্য TTL সহ সাধারণ key। Redis-এর atomicity-ই এগুলোকে concurrency-র অধীনে সঠিক করে তোলে — operation-গুলো interleaving ছাড়াই সম্পন্ন হয়, যা ঠিক একটি rate limiter বা lock-এর যা প্রয়োজন।

### ৩. প্রতিটি deploy-এর পরে cold cache
**আপনি যা দেখেন:** প্রতিটি deploy একটি latency spike এবং একটি database load spike ঘটায়।

**কেন এটি ঘটে:** In-process cache process-এর সাথে সাথেই মারা যায়।

**একটি external cache কীভাবে এটি সমাধান করে:** Cache tier-এর নিজস্ব lifecycle আছে। Application server স্বাধীনভাবে restart, deploy, এবং autoscale করতে পারে; cache পুরো সময় জুড়ে warm থাকে। এই decoupling শুনতে যা মনে হয় তার চেয়ে বেশি মূল্যবান — এটি একটি পুনরাবৃত্ত, স্ব-সৃষ্ট load spike দূর করে।

### ৪. Data structure যা database-এর করা উচিত নয়
**আপনি যা দেখেন:** প্রতিটি page load-এ একটি ব্যয়বহুল `ORDER BY ... LIMIT` দিয়ে পুনরায় গণনা করা একটি leaderboard। Delete-and-insert transaction দিয়ে বজায় রাখা একটি "সম্প্রতি দেখা" তালিকা।

**কেন এটি ঘটে:** একটি relational database-কে একটি general-purpose data structure engine হিসেবে ব্যবহার করা।

**Redis কীভাবে এটি সমাধান করে:** Sorted set স্বাভাবিকভাবেই O(log n) leaderboard দেয়। Lists, sets, hashes, HyperLogLog, এবং streams — প্রতিটি একটি একক দ্রুত operation দিয়ে application logic-এর একটি অংশ প্রতিস্থাপন করে। এখানেই Redis "একটি cache" থাকা বন্ধ করে একটি data structure server হয়ে ওঠে — এবং এটাই মূল কারণ যে কেন এটি Memcached-এর বদলে বেছে নেওয়া হয়, যেটি ইচ্ছাকৃতভাবে সহজতর (শুধু string, multithreaded, ঠিক একটি কাজে চমৎকার)।

## যে মূল্য আপনাকে দিতে হয়

- **একটি network hop।** Local memory nanosecond-এর ব্যাপার; Redis একটি sub-millisecond network round trip। তবুও database-এর চেয়ে বিপুলভাবে দ্রুত, কিন্তু আর বিনামূল্যের নয় — যার মানে হলো এমন কোড যা প্রতি request-এ 50টি cache call করে, সে আসলে তার bottleneck সরায়নি বরং স্থানান্তরিত করেছে (pipelining বা multi-get ব্যবহার করুন)।
- **একটি নতুন critical dependency।** যদি session Redis-এ থাকে এবং Redis down হয়ে যায়, কেউ login করতে পারবে না। এর replication, failover, এবং monitoring দরকার, যেকোনো datastore-এর মতোই।
- **Memory সীমিত এবং eviction বাস্তব।** যখন memory ভরে যায়, key-গুলো policy অনুযায়ী evict হয়ে যায়। যদি আপনি এটিকে একটি durable store হিসেবে ব্যবহার করেন, তাহলে সেটি data loss — এবং Redis-এর persistence option (RDB, AOF) durability *সম্ভব* করে তোলে কিন্তু বিনামূল্যে নয়, এবং database-এর সমতুল্য নয়।
- **Distribution সমস্যা।** একটি cache-কে node জুড়ে shard করা "কোন node এই key ধারণ করে" প্রশ্ন তোলে, এবং সরল modulo hashing cluster পরিবর্তিত হলে সবকিছু নতুন করে সাজিয়ে দেয় — এই কারণেই consistent hashing-এর অস্তিত্ব।
- **এটি চালানোর জন্য আরেকটি system।** Version upgrade, memory tuning, cluster topology, এবং failover behavior — এসবই প্রকৃত কাজ।

## কখন আপনার এটি প্রয়োজন — এবং কখন নয়

| একটি shared cache ব্যবহার করুন যখন | Local cache ঠিক আছে যখন |
|---|---|
| আপনি একাধিক application instance চালান | একক instance, অথবা data immutable |
| State-কে globally consistent হতে হবে (sessions, counters, locks) | Value স্বভাবতই server-local (compiled templates, config) |
| আপনি চান cache deploy-এর পরেও টিকে থাকুক | Sub-microsecond access সত্যিই গুরুত্বপূর্ণ |
| আপনার সমৃদ্ধ data structures প্রয়োজন (Redis) | — |
| Hit rate গুরুত্বপূর্ণ এবং আপনি N-সংখ্যক কপি বহন করতে পারেন না | — |

অনেক পরিণত system উভয়ই চালায়: hottest key-গুলোর জন্য shared cache-এর সামনে একটি ছোট local cache, distribution-এর সবচেয়ে উপরের অংশের জন্য সংক্ষিপ্ত inconsistency মেনে নিয়ে।

## কেন এটি Interview-এ দেখা যায়

Redis বেশিরভাগ design-এ দেখা যায়, তাই শুধু এর নাম বলাটাই সংকেত নয়। প্রকৃত সংকেতগুলো হলো: অভ্যাসবশত নয়, একটি নির্দিষ্ট কারণে (data structures, persistence, Pub/Sub) Redis-কে Memcached-এর বদলে বেছে নেওয়া; কাজের জন্য সঠিক primitive ব্যবহার করা (leaderboard-এর জন্য sorted set, rate limiting-এর জন্য `INCR`, একটি lock-এর জন্য TTL সহ `SETNX`); এবং "Redis down হয়ে গেলে কী হবে?" প্রশ্নের উত্তর নীরবতার চেয়ে ভালো কিছু দিয়ে দেওয়া। বিশেষত Rate limiter এবং leaderboard প্রশ্নগুলো আসলে ছদ্মবেশে Redis data-structure প্রশ্ন।

## এটি কীভাবে সংযুক্ত

এটি **caching strategy**-এর (বিষয় ১৭) একটি বাস্তব রূপ। এটি সেই shared state layer যা **horizontally scaled** stateless server সম্ভব করে তোলে। এটি **rate limiting** (বিষয় ২৫), **distributed locking** (বিষয় ৪০), **WebSockets**-এর পেছনে **Pub/Sub** fan-out, এবং বেশ কয়েকটি **probabilistic data structures**-এর (বিষয় ৪২) জন্য মানক implementation substrate। Cache tier-কে নিজেই scale করার প্রয়োজনই **consistent hashing**-কে (বিষয় ২৪) প্রেরণা দেয়।

**পরবর্তী:** [Message Queues: Kafka vs RabbitMQ](../../Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/why.md) — শুধু cache নয়, বরং কাজকে সময়ের মধ্যে decouple করা।
