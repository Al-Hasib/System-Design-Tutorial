# Distributed Caching with Redis & Memcached

**কঠিনতা স্তর:** Intermediate/Advanced

## শেখার লক্ষ্যসমূহ

- একাধিক application server-এ scale করার সাথে সাথে একটি shared, distributed cache কেন প্রয়োজন হয় তা ব্যাখ্যা করা
- Redis এবং Memcached-কে data structures, persistence, replication, এবং clustering-এর দিক থেকে তুলনা করা
- একটি cache cluster জুড়ে data কীভাবে বিতরণ করা হয় তা বোঝা (উচ্চ স্তরে sharding/consistent hashing)
- Redis-নির্দিষ্ট ফিচারগুলো বর্ণনা করা: data structures, persistence (RDB/AOF), pub/sub, এবং replication
- একটি dedicated distributed cache বনাম per-instance local cache-এর জন্য উপযুক্ত use case চিহ্নিত করা

## স্ক্রিপ্ট

### Hook / ভূমিকা

এতক্ষণ আমরা আলোচনা করেছি কীভাবে আপনার application-এর কাছে data cache করা যায় এবং একটি CDN দিয়ে আপনার ব্যবহারকারীদের কাছে content cache করা যায়। কিন্তু এখন প্রশ্ন হলো: যখন আপনার application আর একটি মাত্র server না হয়ে একটি load balancer-এর পেছনে শখানেক server হয়ে যায়, তখন কী হবে? যদি এই শখানেক server-এর প্রতিটি নিজস্ব local, in-memory cache রাখে, তাহলে আপনার কাছে "cache"-এর শখানেক আলাদা, অসামঞ্জস্যপূর্ণ কপি থাকবে — এবং শতগুণ বেশি cold-start miss হবে। সমাধান হলো একটি **distributed cache**: একটি shared caching layer, যার সাথে প্রতিটি application server নেটওয়ার্কের মাধ্যমে কথা বলে। এই ভিডিওতে, আমরা এই কাজের জন্য সবচেয়ে জনপ্রিয় দুটি technology — Redis এবং Memcached — নিয়ে গভীরে যাব এবং বুঝব কখন কোনটি ব্যবহার করতে হবে।

### কেন আপনার একটি Distributed, Shared Cache প্রয়োজন

দৃশ্যপট কল্পনা করা যাক। একটি "local cache" কল্পনা করুন — আক্ষরিক অর্থে প্রতিটি application server-এর process-এর মেমরিতে বসবাসকারী একটি hash map। এটি অত্যন্ত দ্রুত, কারণ এখানে কোনো network hop নেই। কিন্তু যখনই একাধিক server থাকে, এটি দুইভাবে ভেঙে পড়ে। প্রথমত, consistency: যদি কোনো ব্যবহারকারীর request server A-তে যায়, সেখানে cache হয়, এবং load balancing-এর কারণে তার পরবর্তী request server B-তে যায়, তাহলে server B-এর কোনো ধারণাই নেই যে সেই data আছে — এটা আবার একটা cache miss হয়ে যায়। দ্বিতীয়ত, memory efficiency: আপনি একই data প্রতিটি server-এর মেমরিতে বারবার সংরক্ষণ করছেন, একবার কেন্দ্রীয়ভাবে সংরক্ষণ করার বদলে।

একটি distributed cache এই দুটি সমস্যাই সমাধান করে cache-কে পৃথক application process থেকে সরিয়ে এর নিজস্ব dedicated service-এ স্থানান্তরিত করে — caching server-এর একটি cluster, যার সাথে সব application server নেটওয়ার্কের মাধ্যমে কথা বলে। এখন প্রতিটি application server একই cached data দেখতে পায়, এবং আপনি প্রতিটি data একবারই সংরক্ষণ করেন, cache cluster-এ, আপনি যতগুলো application server-ই scale out করুন না কেন। এর trade-off হলো আপনি একটি network hop যোগ করেছেন, তাই এটি একটি local, in-process cache-এর তুলনায় প্রতি request-এ একটু ধীর, কিন্তু তবুও এটি database-এ hit করার চেয়ে বহুগুণ দ্রুত, এবং এটি consistency ও duplication-এর সমস্যা সমাধান করে যা local cache পারে না।

### Memcached: সহজ, দ্রুত, এবং কেন্দ্রীভূত

**Memcached** হলো সবচেয়ে পুরনো এবং সহজতম distributed caching system-গুলোর একটি, এবং সেই সরলতাই এর প্রধান বিক্রয়যোগ্য বৈশিষ্ট্য। এটি একটি pure key-value store: আপনি একটি key এবং একটি byte-string value ইনপুট দেন, এবং সেই value key দিয়ে ফেরত পান, যতক্ষণ না এর মেয়াদ শেষ হয় বা evict হয়ে যায়। এটাই মূলত সব। Memcached multi-threaded, যা এটিকে multi-core machine-এর সুবিধা চমৎকারভাবে নিতে দেয় সাধারণ operation-এর জন্য অনেক বেশি throughput পাওয়ার জন্য, এবং এটি memory কম পড়লে একটি সরল LRU eviction policy ব্যবহার করে। এতে কোনো built-in persistence নেই — যদি একটি Memcached server restart হয়, তাহলে এর ভেতরের সবকিছু হারিয়ে যায় — এবং কোনো built-in replication নেই; এটি একটি pure, disposable, high-speed cache layer হওয়ার জন্য তৈরি, কোনো কিছুর source of truth হওয়ার জন্য নয়।

### Redis: একটি Data Structure Server

**Redis** একই ধারণা থেকে শুরু হয়েছিল — একটি in-memory key-value cache — কিন্তু এটি আরও শক্তিশালী কিছুতে পরিণত হয়েছে: একটি in-memory data structure server। সাধারণ string ছাড়াও, Redis স্বাভাবিকভাবে lists, sets, sorted sets, hashes, streams, এমনকি geospatial data type সমর্থন করে, যেখানে দক্ষ operation সরাসরি server-এ built-in রয়েছে। এর মানে আপনি এমন জিনিস করতে পারেন যেমন একটি sorted set ব্যবহার করে leaderboard বজায় রাখা, যেখানে Redis নিজেই ranking সামলে নেয়, অথবা একটি lightweight queue হিসেবে একটি list ব্যবহার করা — এমন use case যেগুলোর জন্য Memcached-এর সাধারণ key-value model-এ প্রচুর custom application logic দরকার হতো।

Redis **persistence**-ও সমর্থন করে: এটি পর্যায়ক্রমে তার dataset-এর snapshot নিতে পারে disk-এ (যাকে বলা হয় RDB snapshot) অথবা প্রতিটি write-এর একটি append-only log রাখতে পারে (AOF, Append-Only File) যাতে এটি restart-এর পরও তার data পুনরুদ্ধার করতে পারে — যা Memcached একেবারেই পারে না। Redis **replication** সমর্থন করে, একটি primary হিসেবে এক বা একাধিক read replica সহ চলে, এবং **Redis Cluster** একাধিক node জুড়ে horizontal scaling এবং sharding-এর জন্য built-in high availability সহ। এতে **pub/sub messaging** এবং নির্দিষ্ট কিছু operation-এর উপর atomic transaction-এর মতো অতিরিক্ত ফিচারও আছে।

### Redis বনাম Memcached: কোনটি বেছে নেবেন

তাহলে কোনটি বেছে নেবেন? যদি আপনার প্রয়োজন কেবল "ছোট value সংরক্ষণ করা, দ্রুত সেগুলো ফেরত আনা, উচ্চ throughput, কোনো বাড়তি কিছু নয়" হয়, তাহলে Memcached হালকা, পরিণত, এবং ঠিক সেই কাজে চমৎকার, বিশেষ করে বড় multi-core machine-এ multi-threaded workload-এর জন্য। যদি আপনার সমৃদ্ধ data structures, persistence, high availability-র জন্য replication, pub/sub প্রয়োজন হয়, অথবা আপনি চান আপনার caching layer একটি lightweight message broker বা session store হিসেবেও কাজ করুক, তাহলে Redis আরও বহুমুখী পছন্দ — এবং বাস্তবে, Redis আধুনিক system design-এ বেশি সাধারণভাবে ব্যবহৃত default হয়ে উঠেছে, ঠিক সেই বহুমুখিতার কারণেই, এমনকি অনেক সাধারণ caching পরিস্থিতিতেও।

### একটি Cache Cluster জুড়ে Data বিতরণ করা

আপনি যে technology-ই বেছে নিন না কেন, যখনই আপনার কাছে একটি single cache server-এ ধরে না এমন পরিমাণ data থাকে — অথবা একটি server যা সামলাতে পারে তার চেয়ে বেশি throughput প্রয়োজন হয় — তখন আপনাকে একাধিক cache node জুড়ে data ছড়িয়ে দিতে হবে। এখানেই **sharding** কাজে আসে: প্রতিটি key নির্দিষ্টভাবে একটি node-এ নির্ধারিত হয়, সাধারণত একটি hashing scheme ব্যবহার করে। একটি সরল পদ্ধতি — key-এর hash modulo node-এর সংখ্যা — এর একটি বড় দুর্বলতা আছে: যদি আপনি একটি node যোগ বা বাদ দেন, তাহলে প্রায় প্রতিটি key-এর নির্ধারিত node পরিবর্তিত হয়, যা cache miss-এর একটি বিশাল ঢেউ তৈরি করে। এই কারণেই production system সাধারণত **consistent hashing** ব্যবহার করে, একটি কৌশল যা cluster-এর আকার পরিবর্তনের সময় কতগুলো key স্থানান্তরিত হতে হয় তা কমিয়ে আনে, বিঘ্নকে cache-এর একটি ছোট অংশে সীমাবদ্ধ রাখে, প্রায় পুরোটার বদলে। আমরা Module 6-এ consistent hashing নিয়ে অনেক বিস্তারিতভাবে আলোচনা করব, কিন্তু এখনই জানা ভালো যে এটিই Memcached client-side sharding এবং Redis Cluster-এর internal data distribution-এর পেছনের মানক পদ্ধতি (Redis Cluster প্রযুক্তিগতভাবে node-গুলো জুড়ে বিতরণ করা 16,384 hash slot-এর একটি নির্দিষ্ট সেট ব্যবহার করে, যা একই ধরনের rebalancing সুবিধা অর্জন করে)।

### বাস্তব-জগতের উদাহরণ

একটি e-commerce site-এর কথা ভাবুন একটি flash sale-এর সময়ে। Product inventory count প্রতিনিয়ত পড়া হয় প্রতিটি checkout প্রচেষ্টার মাধ্যমে, ডজনখানেক application server জুড়ে। একটি shared cache ছাড়া, প্রতিটি server-এর একটি stale, locally-cached inventory count থাকতে পারে, যা overselling-এর দিকে নিয়ে যায়। Database-এর সামনে একটি distributed Redis cluster থাকলে, প্রতিটি application server একই shared inventory counter চেক এবং decrement করে, Redis-এর atomic increment/decrement operation ব্যবহার করে, পুরো fleet জুড়ে সংখ্যাগুলো real time-এ সামঞ্জস্যপূর্ণ রাখে, একই সাথে database-কে প্রতিটি page view এবং checkout click-এর আঘাত থেকে বাঁচায়।

আরেকটি ক্লাসিক উদাহরণ: **session storage**। একটি load-balanced web application-এ, একজন ব্যবহারকারীর session data যেকোনো server তার পরবর্তী request handle করুক না কেন, তার কাছে উপলব্ধ থাকতে হবে। প্রতিটি server-এর local memory-র বদলে একটি distributed Redis cache-এ session সংরক্ষণ করার মানে হলো যেকোনো server তাৎক্ষণিকভাবে একজন logged-in ব্যবহারকারীর session পুনরুদ্ধার করতে পারে, যা সত্যিকারের stateless, horizontally scalable application server সম্ভব করে তোলে।

### সংক্ষিপ্ত পুনরালোচনা

চলুন সংক্ষেপে দেখি। একটি single application server-এর বাইরে scale করলে, local in-process cache inconsistency এবং duplication-এর কারণে ভেঙে পড়ে, তাই আপনি একটি distributed cache-এ চলে যান যা সব server নেটওয়ার্কের মাধ্যমে share করে। Memcached একটি সহজ, multi-threaded, pure key-value cache — সরল, high-throughput caching-এর জন্য চমৎকার যেখানে persistence-এর প্রয়োজন নেই। Redis একটি সমৃদ্ধ in-memory data structure server, যা complex type, persistence, replication, এবং clustering সমর্থন করে, যা এটিকে বেশিরভাগ আধুনিক system-এর জন্য আরও নমনীয় default করে তোলে। এবং যখন আপনার data একটি single cache node-এর ধারণক্ষমতা ছাড়িয়ে যায়, তখন sharding — আদর্শভাবে consistent hashing-এর মাধ্যমে — এটিকে একটি cluster জুড়ে ছড়িয়ে দেয় node যোগ বা বাদ দেওয়ার সময় বিঘ্ন কমিয়ে রেখে।

### এরপর কী

আমরা এখন caching-এর একটি সম্পূর্ণ চিত্র তৈরি করেছি — application-level strategy, network edge-এ CDN, এবং Redis ও Memcached-এর মতো distributed cache যা এই সবকিছুকে একটি server fleet জুড়ে একত্রিত করে। এতে Module 4 শেষ হলো। Module 5-এ, আমরা messaging এবং asynchronous system-এ প্রবেশ করব, message queue দিয়ে শুরু করে এবং এই ক্ষেত্রের দুটি বড় নাম তুলনা করে: Kafka এবং RabbitMQ।

## মূল শিক্ষণীয় বিষয়সমূহ

- একটি distributed cache প্রয়োজনীয় হয়ে ওঠে যখনই আপনার একাধিক application server থাকে, কারণ local in-process cache instance-গুলো জুড়ে inconsistency এবং duplicated memory usage তৈরি করে।
- Memcached একটি সহজ, multi-threaded, pure key-value store যাতে কোনো built-in persistence বা replication নেই — কাঁচা speed এবং সরলতার জন্য optimized।
- Redis একটি in-memory data structure server যা সমৃদ্ধ type (lists, sets, sorted sets, hashes), persistence (RDB/AOF), replication, clustering, এবং pub/sub সমর্থন করে — যা এটিকে আরও বহুমুখী করে তোলে।
- Redis Cluster এবং consistent hashing কৌশল cache data-কে একাধিক node জুড়ে shard করতে দেয়, node যোগ বা বাদ দেওয়ার সময় key movement কমিয়ে রেখে।
- সাধারণ distributed cache use case-এর মধ্যে রয়েছে database query caching, session storage, rate limiting counter, leaderboard, এবং real-time counter (যেমন, inventory)।
- সহজ, সর্বোচ্চ-throughput key-value caching-এর জন্য Memcached বেছে নিন; যখন আপনার সমৃদ্ধ data structures, durability, বা high-availability replication প্রয়োজন তখন Redis বেছে নিন।
