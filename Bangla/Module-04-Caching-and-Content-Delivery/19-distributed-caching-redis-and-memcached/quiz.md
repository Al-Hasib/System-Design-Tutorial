# অনুশীলন ও ইন্টারভিউ প্রশ্ন

**১. একাধিক application server-এ scale করলে একটি local, in-process cache কেন ভালোভাবে কাজ করা বন্ধ করে দেয়?**
একটি local cache একটি একক server-এর মেমরিতে থাকে, তাই load balancer-এর পেছনের প্রতিটি server নিজস্ব একটি পৃথক, অসামঞ্জস্যপূর্ণ "cache"-এর কপি তৈরি করে, যা fleet জুড়ে পুনরাবৃত্ত cache miss এবং duplicated memory usage ঘটায়। একটি distributed cache এটি সমাধান করে প্রতিটি application server-কে নেটওয়ার্কের মাধ্যমে cached data-এর একটি একক shared দৃশ্য দিয়ে।

**২. Redis এবং Memcached-এর মধ্যে তিনটি মূল পার্থক্য উল্লেখ করুন।**
Redis সমৃদ্ধ data structures (lists, sets, sorted sets, hashes) সমর্থন করে, অন্যদিকে Memcached শুধুমাত্র সাধারণ key-value byte string সমর্থন করে। Redis built-in persistence (RDB/AOF) এবং replication দেয়, যেখানে Memcached-এ কোনোটিই নেই। Redis-এ pub/sub এবং Lua scripting-এর মতো অতিরিক্ত ফিচারও আছে, যেখানে Memcached একটি ন্যূনতম, multi-threaded pure cache।

**৩. কখন আপনি Redis-এর বদলে Memcached বেছে নেবেন?**
Memcached বেছে নিন যখন আপনার প্রয়োজন কঠোরভাবে সাধারণ, high-throughput key-value caching, যেখানে persistence, replication, বা complex data structures-এর কোনো প্রয়োজন নেই — এর multi-threaded architecture সরল get/set workload-এ প্রতি node-এ চমৎকার raw throughput দিতে পারে ন্যূনতম operational জটিলতায়।

**৪. একটি cache cluster shard করার সময় consistent hashing কোন সমস্যা সমাধান করে?**
সরল modulo hashing (hash(key) % N)-এর ক্ষেত্রে, একটি একক node যোগ বা বাদ দিলে প্রায় প্রতিটি key-এর নির্ধারিত node পরিবর্তিত হয়, যা cache miss-এর একটি বিশাল ঢেউ ঘটায়। Consistent hashing key এবং node-গুলোকে একটি hash ring-এ map করে, যাতে cluster-এর আকার পরিবর্তিত হলে শুধু অল্প কিছু key-ই স্থানান্তরিত হওয়া প্রয়োজন হয়, বিঘ্ন কমিয়ে।

**৫. Redis Cluster কীভাবে node জুড়ে data বিতরণ করে তা ব্যাখ্যা করুন।**
Redis Cluster keyspace-কে নির্দিষ্ট 16,384টি hash slot-এ ভাগ করে, এবং cluster-এর প্রতিটি node সেই slot-গুলোর একটি উপসেট নিজের দখলে রাখে। যখন node যোগ বা বাদ দেওয়া হয়, তখন শুধুমাত্র প্রভাবিত slot (এবং তাদের key) পুনর্নির্ধারিত ও স্থানান্তরিত হয়, পুরো keyspace rehash করার বদলে।

**৬. একটি horizontally scaled web application-এ ব্যবহারকারীর session সংরক্ষণের জন্য Redis কেন একটি সাধারণ পছন্দ?**
একটি shared Redis cache-এ session সংরক্ষণ করার মানে হলো যেকোনো application server একজন ব্যবহারকারীর session data পুনরুদ্ধার করতে পারে, নির্দিষ্ট request কোন server handle করছে তা নির্বিশেষে, যেহেতু load balancer পরপর request বিভিন্ন server-এ route করতে পারে। এটি সত্যিকারের stateless application server সম্ভব করে তোলে যা session continuity না হারিয়ে স্বাধীনভাবে scale up বা down করা যায়।

**৭. Memcached server process restart হলে data-এর কী হয়? এটি Redis থেকে কীভাবে আলাদা?**
Memcached-এ কোনো persistence নেই, তাই restart হলে সমস্ত cached data হারিয়ে যায় — এটি গ্রহণযোগ্য কারণ Memcached নিছক একটি disposable cache হওয়ার জন্য তৈরি, কোনো source of truth নয়। Redis ঐচ্ছিকভাবে RDB snapshots বা একটি AOF log-এর মাধ্যমে disk-এ data persist করতে পারে, যা এটিকে restart-এর পরে তার dataset পুনরায় লোড করতে দেয়, যদিও এটি এখনও সাধারণত ভালো অভ্যাস যে Redis-কেও একটি প্রকৃত source of truth দ্বারা সমর্থিত cache হিসেবে বিবেচনা করা।

**৮. একটি বাস্তব-জগতের use case বর্ণনা করুন যেখানে Redis-এর data structures (সাধারণ key-value ছাড়াও) Memcached-এর তুলনায় একটি স্পষ্ট সুবিধা দেয়।**
একটি leaderboard ফিচার একটি Redis sorted set ব্যবহার করতে পারে, যেখানে Redis স্বাভাবিকভাবেই ranked ordering বজায় রাখে এবং সরাসরি server-এ দক্ষ range query সমর্থন করে (যেমন, "top 10" বা "user's rank")। Memcached-এর সাধারণ key-value model দিয়ে একই ফিচার implement করতে application-এ data fetch এবং পুনরায় sort করতে হবে, যা অনেক কম দক্ষ।

**৯. একটি distributed cache যোগ করলে local in-process cache-এর তুলনায় request latency profile কীভাবে পরিবর্তিত হয়, এবং এটি তবুও কেন সাধারণত মূল্যবান?**
একটি distributed cache একটি network round trip যোগ করে যা একটি local in-process cache-এ থাকে না, তাই এটি একটি in-process lookup-এর তুলনায় প্রতি request-এ কিছুটা ধীর। এটি তবুও মূল্যবান কারণ সেই বাড়তি latency (sub-millisecond থেকে কয়েক millisecond) database-এ hit করার latency-র তুলনায় বিপুলভাবে ছোট, একই সাথে এটি সেই consistency এবং memory-duplication সমস্যাগুলো সমাধান করে যা local cache পারে না।

**১০. অনেক application server জুড়ে shared inventory count সহ একটি flash-sale পরিস্থিতিতে, atomic operation (যেমন Redis-এর INCR/DECR) সহ একটি distributed cache কেন গুরুত্বপূর্ণ?**
একই সাথে checkout প্রক্রিয়াকরণকারী একাধিক application server-কে একই inventory count real time-এ দেখতে এবং আপডেট করতে হবে; একটি shared cache-এ atomic increment/decrement operation নিশ্চিত করে যে concurrent decrement race করবে না বা ভুল count তৈরি করবে না (যা overselling ঘটাতে পারে)। প্রতিটি server-এর একটি local cache প্রতিটি server-কে inventory-এর একটি stale, স্বাধীন দৃশ্য দেবে, যা overselling-কে অনেক বেশি সম্ভাব্য করে তুলবে।

**১১. Redis pub/sub কী, এবং একটি system-এ caching-এর পাশাপাশি এটি কীভাবে ব্যবহৃত হতে পারে?**
Redis pub/sub client-দের নামযুক্ত channel-এ message publish করতে দেয় এবং অন্য client-রা সেগুলো real time-এ গ্রহণ করতে subscribe করে। এটি প্রায়ই caching-এর পাশাপাশি ব্যবহৃত হয় যেমন একাধিক application server-এ cache-invalidation event broadcast করার জন্য, অথবা সাধারণ ক্ষেত্রে একটি পৃথক messaging system ছাড়াই lightweight real-time notification-এর জন্য।

**১২. যদি আপনার একটি দ্রুত cache এবং durability guarantee উভয়ই প্রয়োজন হয় যাতে cached computation result restart-এর পরেও টিকে থাকে, তাহলে এই ভিডিও থেকে কোন technology আপনি বেছে নেবেন এবং কেন?**
Redis, কারণ এটি persistence mechanism (RDB snapshots এবং/অথবা AOF logging) সমর্থন করে যা এটিকে restart-এর পরে তার dataset পুনরায় লোড করতে দেয়, Memcached-এর বিপরীতে যার কোনো persistence নেই এবং restart-এ সবকিছু হারায়।
