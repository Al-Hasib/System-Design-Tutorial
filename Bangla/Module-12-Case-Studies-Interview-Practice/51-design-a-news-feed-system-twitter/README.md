# একটি নিউজ ফিড সিস্টেম ডিজাইন করা (Twitter/Facebook-এর মতো)

**কঠিনতা:** Advanced (Capstone)
**আনুমানিক দৈর্ঘ্য:** 25-30 মিনিট
**পূর্বশর্ত:**
- [Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md)
- [Caching Strategies and Cache Invalidation](../../Module-04-Caching-and-Content-Delivery/17-caching-strategies-and-cache-invalidation/README.md)
- [Distributed Caching: Redis and Memcached](../../Module-04-Caching-and-Content-Delivery/19-distributed-caching-redis-and-memcached/README.md)
- [Message Queues: Kafka vs RabbitMQ](../../Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/README.md)
- [Publish-Subscribe Pattern](../../Module-05-Messaging-and-Asynchronous-Systems/21-publish-subscribe-pattern/README.md)
- [Batch vs Stream Processing](../../Module-05-Messaging-and-Asynchronous-Systems/23-batch-vs-stream-processing/README.md)
- [Consistent Hashing Explained](../../Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/README.md)

## শেখার লক্ষ্যসমূহ (Learning Objectives)

- একটি "design a news feed" প্রম্পটের জন্য পুরোপুরি একটি মক ইন্টারভিউ চালানো — clarifying questions থেকে শুরু করে একটি defensible high-level architecture পর্যন্ত।
- একটি Twitter/Facebook-স্টাইলের feed সিস্টেমের স্কেল পরিমাপ করা: daily active users, posts/sec, এবং fan-out writes/sec।
- fan-out-on-write, fan-out-on-read, এবং hybrid delivery strategy তুলনা করা, এবং celebrity account-এর জন্য কোনটি ব্যবহার করা উচিত তা যুক্তি দিয়ে প্রমাণ করা।
- hot-key/celebrity সমস্যা সমাধানের জন্য আগের মডিউলগুলো থেকে consistent hashing, sharding, এবং caching প্যাটার্ন প্রয়োগ করা।
- একটি বাস্তব feed সিস্টেমে write cost, read latency, এবং eventual consistency-এর মধ্যে trade-off স্পষ্টভাবে ব্যাখ্যা করা।

## স্ক্রিপ্ট (Script)

### শুরু / ভূমিকা (Hook / Intro)

Twitter, Facebook/Meta, LinkedIn, এবং Instagram-এর মতো কোম্পানিতে "design a news feed" সবচেয়ে সাধারণ capstone প্রশ্নগুলোর একটি — এবং এটি জনপ্রিয় ঠিক এই কারণেই যে এটি আপনাকে একটি system design কোর্সের প্রায় সবকিছু একটি উত্তরে একত্র করতে বাধ্য করে: sharded storage, caching, pub-sub messaging, এবং consistency বনাম latency-এর মতো distributed systems trade-off।

এই ভিডিওতে আমরা এটি একটি বাস্তব ইন্টারভিউয়ের মতো করে চালাব: requirements পরিষ্কার করা, বাস্তব সংখ্যা দিয়ে সিস্টেমের আকার নির্ধারণ করা, একটি high-level architecture স্কেচ করা, এবং তারপর সত্যিকার গুরুত্বপূর্ণ দুই-তিনটি কম্পোনেন্টে গভীরে যাওয়া — fan-out strategy এবং celebrity problem। আমরা স্পষ্টভাবে এই কোর্সের আগের মডিউলগুলোর ধারণা পুনরায় ব্যবহার করব: Module 3 থেকে sharding, Module 4 থেকে caching, Module 5 থেকে message queue ও pub-sub, এবং Module 6 থেকে consistent hashing। যদি আপনি সেগুলো এখনও না দেখে থাকেন, ঠিক এই ধরনের প্রশ্নেই সেগুলোর সুফল পাওয়া যায়।

### ধাপ ১: Requirements পরিষ্কার করা

ডিজাইন শুরু করার আগে scope পরিষ্কার করা কখনো এড়িয়ে যাবেন না। এই সেশনের জন্য আমি যা জিজ্ঞাসা করব এবং যে উত্তর ধরে নেব তা এখানে দেওয়া হলো।

**Functional requirements:**
- ব্যবহারকারীরা post তৈরি করতে পারবে (text, ঐচ্ছিকভাবে media) — এটি write path।
- ব্যবহারকারীরা অন্য ব্যবহারকারীদের follow/unfollow করতে পারবে — একটি directed এবং asymmetric follow graph (আমি আপনাকে follow করতে পারি, আপনি আমাকে ফিরে follow না করলেও)।
- ব্যবহারকারীদের একটি home timeline ("news feed") থাকবে, যেখানে তারা যাদের follow করে তাদের সবার post দেখানো হবে, reverse-chronological অথবা ranked আকারে।
- feed টি ranked, বিশুদ্ধভাবে chronological নয় — recency, engagement signal, এবং poster-এর সাথে affinity — সবকিছু বিবেচনায় আসে।

আমি স্পষ্টভাবে scope-এর বাইরে রাখব: comments, likes, direct messages, এবং ad injection — এগুলো প্রত্যেকটি নিজেই আলাদা একটি ইন্টারভিউ প্রশ্ন হতে পারে এমন আলাদা সিস্টেম।

**Non-functional requirements:**
- **Low read latency** — home timeline লোড করার জন্য। ব্যবহারকারীরা আশা করে এটি এক সেকেন্ডের অনেক কম সময়ে খুলবে। Reads, writes-এর তুলনায় বহুগুণ বেশি, তাই আমরা write complexity-এর বিনিময়ে read latency অপ্টিমাইজ করি।
- **Eventual consistency গ্রহণযোগ্য।** যদি একটি নতুন post একজন follower-এর feed-এ দেখাতে কয়েক সেকেন্ড সময় নেয়, সেটি ঠিক আছে। আমরা স্পষ্টভাবে একটি strongly consistent সিস্টেম তৈরি করছি না।
- **উচ্চ write fan-out** সাবলীলভাবে সামলাতে হবে। একজন celebrity account-এর একটি মাত্র post কোটি কোটি follower-এর কাছে পৌঁছাতে হতে পারে — এটিই সমস্যার মূল কেন্দ্রবিন্দু এবং ইন্টারভিউয়ে বেশিরভাগ সিগন্যাল এখান থেকেই আসে।
- সিস্টেমকে highly available এবং horizontally scalable হতে হবে; consistency-এর চেয়ে availability-কে প্রাধান্য দেওয়া হয় (CAP-এর ভাষায় একটি AP-ঘেঁষা সিস্টেম)।

### ধাপ ২: Capacity Estimation

চলুন এতে বাস্তব সংখ্যা বসাই, যাতে আমাদের architecture-এর সিদ্ধান্তগুলো বাস্তবতার ভিত্তিতে হয়।

- **৩০ কোটি (300 million) daily active users (DAU)।**
- গড়ে একজন ব্যবহারকারী দিনে দুইবার post করে → দিনে **৬০ কোটি (600 million) posts**, যা প্রায় **~৭,০০০ posts/sec গড়ে**, এবং ধরে নিচ্ছি peak traffic গড়ের ৩ গুণ, তাই **peak-এ ~২১,০০০ posts/sec**।
- গড় follower সংখ্যা প্রতি ব্যবহারকারীতে **৫০০ জন** — এটি "normal" ক্ষেত্র।
- কিন্তু একটি long tail আছে: প্রায় **১০ লাখ (1 million) celebrity/influencer account-এর প্রত্যেকের ১ কোটি (10 million)-এর বেশি follower আছে**, কারো কারো ৫-১০ কোটি পর্যন্ত।
- একটি feed সিস্টেমের জন্য read-to-write ratio সাধারণত **100:1 থেকে 1000:1** হয় — প্রতিটি ব্যবহারকারী post করার চেয়ে অনেক বেশি বার তাদের feed চেক করে। ৩০ কোটি DAU দিনে প্রায় ১০ বার feed চেক করলে, তা হয় দিনে **৩০০ কোটি (3 billion) feed reads**, অর্থাৎ গড়ে প্রায় **৩৫,০০০ reads/sec**, peak-এ আরো বেশি।
- **Fan-out writes/sec**: যদি আমরা naive-ভাবে প্রতিটি post write-এর সময়ই প্রতিটি follower-এর timeline-এ fan out করি, গড় ক্ষেত্রে তা হবে 7,000 posts/sec × 500 followers = **৩৫ লাখ (3.5 million) fan-out writes/sec**, শুধু সাধারণ ব্যবহারকারীদের থেকেই। ৫ কোটি follower-বিশিষ্ট একজন celebrity-র একটি মাত্র post একাই ৫ কোটি fan-out write একসাথে সৃষ্টি করে — এই একটি event সিস্টেম-ব্যাপী steady-state load-কেও ছাড়িয়ে যেতে পারে।
- **Precomputed timeline cache-এর জন্য storage**: যদি আমরা প্রতিটি ব্যবহারকারীর home timeline-এর সর্বশেষ ~৮০০টি post ID cache করি (একটি post ID ~৮ বাইট, প্লাস metadata — ধরি এন্ট্রি প্রতি ১০০ বাইট বাজেট), তাহলে তা হয় 300M users × 800 entries × 100 bytes ≈ **২৪ TB** cache data। এটি অনেক বেশি, এবং এটি সরাসরি বলে দেয় যে আমাদের একটি distributed cache cluster দরকার, একটি একক Redis box নয়।

এই সংখ্যাগুলো ইতিমধ্যে আমাদের দুটি বিষয় বলে দেয়: fan-out হলো প্রধান write cost, এবং আমাদের এমন একটি caching layer দরকার যার আকার হবে কয়েক দশ terabyte, বহু নোডে sharded।

### ধাপ ৩: High-Level Design

উচ্চ পর্যায়ে, request flow দেখতে এরকম হয়:

**Client → Load Balancer → Post Service / Feed Service।**

- **Post Service** post তৈরি করার কাজ সামলায়: এটি post যাচাই করে, একটি **sharded post store**-এ লেখে (একটি database যা, ধরা যাক, user ID অথবা post ID অনুসারে consistent hashing ব্যবহার করে partition করা — সরাসরি Module 3 এবং Module 6 থেকে), এবং তারপর একটি "new post" event একটি **message queue / pub-sub সিস্টেমে** publish করে (এখানে Kafka স্বাভাবিক পছন্দ, Module 5 অনুযায়ী)।
- একটি **Fan-out Service** সেই event stream-এ subscribe করে। প্রতিটি নতুন post-এর জন্য, এটি **follow-graph store**-এ লেখকের followers খুঁজে বের করে এবং post ডেলিভারির পদ্ধতি ঠিক করে: হয় তাৎক্ষণিকভাবে followers-এর precomputed timeline-এ push করে (fan-out-on-write), অথবা কিছু না করে পরে pull হওয়ার জন্য রেখে দেয় (fan-out-on-read) — এই বিষয়ে আরো বিস্তারিত deep dive অংশে।
- একটি **Ranking Service** candidate posts-কে স্কোর দেয় (recency, affinity, predicted engagement) — কৌশল অনুযায়ী fan-out সময়ে অথবা read সময়ে।
- একটি **Cache layer** (Redis cluster, consistent hashing দ্বারা sharded) precomputed home timeline-গুলো user ID দিয়ে key করা sorted sets হিসেবে সংরক্ষণ করে, যাতে **Feed Service** পুরো follow graph জুড়ে একটি fan-out query চালানোর বদলে কয়েকটি মাত্র cache read দিয়ে "get my timeline" request সার্ভ করতে পারে।
- **Follow-Graph Store** হলো একটি আলাদা service/database যা "user X কাকে follow করে" এবং "user Y-কে কারা follow করে" এই ধরনের graph query-এর জন্য অপ্টিমাইজ করা — এর নিজস্ব indexing strategy দরকার, কারণ celebrity-দের follower list বিশাল।

তাহলে read path হলো: Client → LB → Feed Service → Redis (precomputed timeline) → post store/CDN থেকে post content hydrate করা → rank/merge → return। Write path হলো: Client → LB → Post Service → sharded post DB + event publish → Fan-out Service (queue-এর মাধ্যমে) → cache আপডেট।

### ধাপ ৪: মূল কম্পোনেন্টগুলোতে গভীরে যাওয়া (Deep Dive)

**৪ক. Fan-out-on-write বনাম fan-out-on-read (Module 5: queues এবং pub-sub)।**
Fan-out-on-write (push model) মানে: যখন একটি post তৈরি হয়, আমরা তাৎক্ষণিকভাবে প্রতিটি follower-এর precomputed timeline cache-এ তার একটি reference push করি। Reads সস্তা হয়ে যায় — শুধু cached list fetch করলেই হয়। কিন্তু writes ব্যয়বহুল এবং বার্স্টি (bursty) হয়ে ওঠে, ঠিক যেমনটা আমরা celebrity account-এ দেখেছি যা কোটি কোটি fan-out operation তৈরি করে। আমরা follower list-এর প্রতিটি shard-এর জন্য একটি করে Kafka topic ব্যবহার করি, যাতে fan-out worker-রা সমান্তরালে consume করতে পারে এবং Post Service থেকে decoupled থাকে — যদি fan-out ধীর হয়ে যায়, তাও posting সফল হবে; queue backpressure শুষে নেয়।

Fan-out-on-read (pull model) মানে: আমরা write-এর সময় কিছুই push করি না। বরং, যখন একজন ব্যবহারকারী তার feed খোলে, তখন Feed Service তাদের follow করা সবার post-এর উপর query চালায়, on demand, এবং real-time-এ merge/rank করে। এতে writes অত্যন্ত সস্তা হয়ে যায় — post store-এ একটি মাত্র write — কিন্তু reads ব্যয়বহুল হয়ে ওঠে, কারণ ৫০০ জনকে follow করা একজন ব্যবহারকারীর প্রতিবার feed খোলার সময় ৫০০টি lookup merge ও rank করতে হয়।

**৪খ. Celebrity/hot-key সমস্যা (Module 6: consistent hashing) — একটি hybrid পদ্ধতি।**
আমাদের স্কেলে কোনো pure strategy-ই ভালো কাজ করে না। প্রমিত production উত্তর, এবং Twitter নিজেই যা প্রকাশ্যে বর্ণনা করেছে, তা হলো একটি **hybrid** পদ্ধতি: বেশিরভাগ ব্যবহারকারীর জন্য (গড়ে ৫০০ follower — push করা সস্তা) fan-out-on-write ব্যবহার করা, কিন্তু একটি follower threshold-এর উপরে থাকা celebrity account-এর জন্য write-time fan-out সম্পূর্ণ এড়িয়ে গিয়ে বরং তাদের post read সময়ে merge করা। যখন একজন সাধারণ ব্যবহারকারী তার feed খোলে, Feed Service তাদের precomputed timeline cache থেকে পড়ে (দ্রুত) এবং আলাদাভাবে তারা যে অল্প সংখ্যক celebrity follow করে তাদের সাম্প্রতিক post fetch করে (এটিও দ্রুত, কারণ একজন ব্যবহারকারী মাত্র হাতেগোনা কয়েকজন celebrity-কে follow করে, যদিও প্রতিটি celebrity-র লক্ষ লক্ষ follower থাকতে পারে), এবং rank করার আগে দুটি সোর্স merge করে।

এটি caching layer-এও একটি hot-key সমস্যা: একজন celebrity-র post ID কয়েক সেকেন্ডের মধ্যে লক্ষ লক্ষ client দ্বারা read হতে পারে। Redis cluster জুড়ে consistent hashing key-গুলোকে সমানভাবে node-এ বিতরণ করে এবং পুরো cache reshuffle ছাড়াই node যোগ করতে দেয়, কিন্তু সত্যিকারের একটি hot key-এর জন্য আমরা তারপরও সেই নির্দিষ্ট key-এর সামনে read replica অথবা local/CDN-edge caching যোগ করি, যাতে একটি single shard-এ ওভারলোড না হয়।

**৪গ. Precomputed timeline caching (Module 4)।**
আমরা একটি cache-aside প্যাটার্ন ব্যবহার করি: precomputed timeline Redis-এ প্রতি ব্যবহারকারীতে post ID-এর একটি sorted set হিসেবে থাকে, rank/time অনুযায়ী স্কোর করা। একটি cache miss হলে (যেমন, একদম নতুন ব্যবহারকারী, বা evict হওয়া key), Feed Service follow graph এবং post store থেকে timeline পুনর্নির্মাণে fall back করে, তারপর cache পুনরায় পূরণ করে। TTL এবং সীমিত list দৈর্ঘ্য (যেমন, শুধু সর্বশেষ ~৮০০টি এন্ট্রি) আমাদের ~২৪ TB অনুমান বিবেচনায় memory-কে সীমাবদ্ধ রাখে।

**৪ঘ. Post store shard করা (Module 3)।**
Posts author ID (বা তার একটি hash) দিয়ে sharded, যাতে একজন নির্দিষ্ট ব্যবহারকারীর নিজের post read/write একটি shard-এই থাকে, এবং আমরা consistent hashing ব্যবহার করি যাতে ক্যাপাসিটি বাড়ানোর সময় resharding-এর ঝামেলা কমে যায় — যে একই কৌশল আমাদের cache layer-কেও রক্ষা করে।

### ধাপ ৫: Bottleneck এবং Trade-off

- **Celebrity-দের জন্য hot shard / hot key**: hybrid fan-out থাকা সত্ত্বেও, একজন celebrity-র follow-graph entry এবং post-store row একটি hot key হয়ে উঠতে পারে। এটি read replica, request coalescing, এবং edge caching দিয়ে প্রশমিত করা যায়।
- **Fan-out-on-write cost বনাম fan-out-on-read latency**: push হলো write-heavy এবং storage-heavy; pull হলো read-heavy এবং latency-heavy। Hybrid পদ্ধতি worst-case write cost বাউন্ড করার বিনিময়ে read-path-এ সামান্য complexity যোগ করে।
- **Ranking complexity বনাম latency**: একটি অধিক পরিশীলিত ML ranking model relevance উন্নত করে কিন্তু read path-এ compute time যোগ করে; আমরা fan-out সময়ে rough ranking precompute করে এবং read সময়ে lightweight re-ranking করে এটি ভারসাম্য করতে পারি।
- **Eventual consistency**: followers একটি post কয়েক সেকেন্ড দেরিতে দেখতে পারে, অথবা বিরল ক্ষেত্রে সংক্ষিপ্তভাবে ভুল ক্রমে — আমাদের non-functional requirements বিবেচনায় এটি একটি গ্রহণযোগ্য trade-off।

### সংক্ষিপ্তসার (Recap)

আমরা functional এবং non-functional requirements পরিষ্কার করেছি, ৩০ কোটি DAU এবং লক্ষ লক্ষ celebrity follower-সহ সিস্টেমের আকার নির্ধারণ করেছি, Post Service → queue/pub-sub → Fan-out Service → cached timeline-এর একটি pipeline ডিজাইন করেছি, এবং consistent hashing ও cache-aside precomputed timeline দ্বারা সমর্থিত একটি hybrid fan-out strategy দিয়ে celebrity hot-key সমস্যার সমাধান করেছি।

### এরপর কী (What's Next)

এই ডিজাইনটি প্রসারিত করার চেষ্টা করুন: আপনি কীভাবে real-time notification যোগ করবেন, ইতিমধ্যে fan-out হয়ে যাওয়া cache-এ post edit/delete propagate করার সমর্থন দেবেন, অথবা Module 5-এর batch-vs-stream trade-off ব্যবহার করে ranking-কে একটি streaming pipeline-এ সরাবেন? এই ফোল্ডারের quiz ফাইলে এরকম বেশ কয়েকটি follow-up প্রশ্নের model answer সহ আলোচনা করা হয়েছে।

## মূল শিক্ষণীয় বিষয় (Key Takeaways)

- ডিজাইন শুরু করার আগে সবসময় functional requirements (post, follow, timeline, rank) কে non-functional requirements (latency, consistency, fan-out scale) থেকে আলাদা করুন।
- আপনার architecture-কে বাস্তব capacity সংখ্যায় ভিত্তি দিন — DAU, posts/sec, follower distribution — কারণ এগুলোই নির্ধারণ করে কোন trade-off গুরুত্বপূর্ণ।
- Fan-out-on-write writes-এর মূল্যে reads অপ্টিমাইজ করে; fan-out-on-read reads-এর মূল্যে writes অপ্টিমাইজ করে; একটি hybrid পদ্ধতি celebrity account-গুলোকে আলাদাভাবে route করে দুটোরই সেরা দিক পায়।
- Consistent hashing (Module 6) হলো সেই common টুল যা sharded post store এবং distributed cache — উভয়কেই hot key ও অসম load থেকে রক্ষা করে।
- Cache-aside precomputed timeline (Module 4), pub-sub-চালিত fan-out (Module 5), এবং sharded storage (Module 3) — এই তিনটি একসাথে একটি বাস্তব-জগতের feed সিস্টেমের মেরুদণ্ড গঠন করে।
- Eventual consistency এখানে একটি স্পষ্ট, গ্রহণযোগ্য ডিজাইন সিদ্ধান্ত — requirements-এ যা দাবি করা হয়নি সেই strong consistency-এর জন্য অতিরিক্ত ইঞ্জিনিয়ারিং করবেন না।
