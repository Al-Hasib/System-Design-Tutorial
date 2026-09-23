# একটি ভিডিও স্ট্রিমিং প্ল্যাটফর্ম ডিজাইন করা (YouTube/Netflix-এর মতো)

**কঠিনতা:** Advanced (Capstone)। **প্রয়োজনীয় পূর্বজ্ঞান:** [Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md), [Caching Strategies and Cache Invalidation](../../Module-04-Caching-and-Content-Delivery/17-caching-strategies-and-cache-invalidation/README.md), [CDN Explained](../../Module-04-Caching-and-Content-Delivery/18-cdn-explained/README.md), [Message Queues: Kafka vs RabbitMQ](../../Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/README.md), [Batch vs Stream Processing](../../Module-05-Messaging-and-Asynchronous-Systems/23-batch-vs-stream-processing/README.md), [Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md), [Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md)

## শেখার লক্ষ্য (Learning Objectives)

- একটি বড় মাপের ভিডিও স্ট্রিমিং প্ল্যাটফর্মের জন্য সিস্টেম ডিজাইন ইন্টারভিউকে requirements থেকে trade-offs পর্যন্ত কাঠামোবদ্ধ করা।
- YouTube/Netflix স্কেলে storage, transcoding, এবং CDN egress bandwidth অনুমান (estimate) করা।
- এমন একটি asynchronous, queue-driven transcoding pipeline ডিজাইন করা যা upload-কে processing থেকে decouple করে।
- ব্যাখ্যা করা কীভাবে CDN, edge caching, এবং adaptive bitrate streaming (HLS/DASH) একত্রে কাজ করে বিশ্বব্যাপী low-latency, high-availability playback দেওয়ার জন্য।
- ভিডিও প্ল্যাটফর্মের প্রধান bottleneck গুলো (viral thundering herd, long-tail storage cost, transcoding cost/latency) চিহ্নিত করা এবং trade-off নিয়ে যুক্তি দেওয়া।

## স্ক্রিপ্ট (Script)

### Hook / ভূমিকা

প্রতি মিনিটে, YouTube-এর মতো প্ল্যাটফর্মে শত শত ঘণ্টার ভিডিও আপলোড হয়। এই প্রতিটি আপলোডকে অর্ধ ডজন ফরম্যাটে transcode করতে হয়, টেকসইভাবে (durably) সংরক্ষণ করতে হয়, এবং বিশ্বজুড়ে কোটি কোটি ডিভাইসে সাথে সাথে স্ট্রিম করার উপযোগী করতে হয় — 3G-তে থাকা ফোন, গিগাবিট ফাইবারে থাকা স্মার্ট TV, এয়ারপোর্ট Wi-Fi-তে থাকা ল্যাপটপ — কোনো buffering ছাড়াই। এটি সবচেয়ে জনপ্রিয় capstone ইন্টারভিউ প্রশ্নগুলোর একটি, কারণ এটি এই কোর্সের প্রায় প্রতিটি মডিউলকে স্পর্শ করে: storage ও sharding, caching ও CDN, message queue, load balancing, এবং scalability-র মূল ধারণাগুলো একটি সিস্টেমে একসাথে আসে। আজ আমরা YouTube বা Netflix-এর মতো একটি ভিডিও স্ট্রিমিং প্ল্যাটফর্ম প্রথম থেকে শেষ পর্যন্ত ডিজাইন করব, ঠিক যেভাবে একটি বাস্তব onsite ইন্টারভিউতে করা হয়।

### ধাপ ১: Requirements স্পষ্ট করা

একটি বাক্সও আঁকার আগে, আমরা ইন্টারভিউয়ারের সাথে scope স্পষ্ট করে নিই।

**Functional requirements:**
- ব্যবহারকারীরা বিভিন্ন আকার ও ফরম্যাটের একটি ভিডিও ফাইল **আপলোড** করতে পারবে।
- প্ল্যাটফর্ম প্রতিটি আপলোডকে একাধিক resolution এবং bitrate-এ **transcode** করবে (যেমন, 240p, 360p, 480p, 720p, 1080p, 4K) যাতে ক্লায়েন্টরা তাদের নেটওয়ার্ক অবস্থার সাথে খাপ খাওয়াতে পারে।
- ব্যবহারকারীরা **adaptive bitrate streaming** সহ ভিডিও **স্ট্রিম/প্লেব্যাক** করতে পারবে — bandwidth পরিবর্তনের সাথে সাথে player নিরবচ্ছিন্নভাবে quality পরিবর্তন করবে।
- মৌলিক metadata অপারেশন: title, description, thumbnail, view count, likes।

**স্পষ্টভাবে scope-এর বাইরে** (ইন্টারভিউতে এটি জোরে বলুন — এটি judgment দেখায়): full-text search ranking, recommendation সিস্টেম, comments/social graph, live streaming ingest (এটি trade-offs অংশে সংক্ষেপে উল্লেখ করব), monetization/ads। আমরা উল্লেখ করব যে এগুলো আছে, কিন্তু আমাদের ডিজাইনের সময় upload → transcode → store → deliver → play-এর উপর কেন্দ্রীভূত রাখব।

**Non-functional requirements:**
- **High availability** — ভিডিও প্লেব্যাকই মূল প্রোডাক্ট; read path-কে আঞ্চলিক (regional) failure সহ্য করতে হবে।
- **কম startup latency এবং সর্বনিম্ন buffering** — ব্যবহারকারীরা আশা করে প্লেব্যাক ১-২ সেকেন্ডের মধ্যে শুরু হবে এবং rebuffer খুব কম ঘটবে।
- **বিশাল read-heavy স্কেল** — read (views) write (uploads)-এর তুলনায় বহুগুণ বেশি, হয়তো 1000:1 বা তার বেশি অনুপাতে।
- **Durability** — একবার কোনো ভিডিও গ্রহণ করা হলে, সেটি কখনো হারানো যাবে না; view counter-এ eventual consistency চলবে, কিন্তু আসল ভিডিও বাইটগুলোর জন্য শক্তিশালী durability গ্যারান্টি প্রয়োজন (যেমন, 11 nines, S3-এর মতো object storage-এর সমতুল্য)।

### ধাপ ২: Capacity Estimation

চলুন YouTube-স্কেলে বাস্তব সংখ্যা বসাই।

**Uploads:** ধরা যাক প্রতি মিনিটে ~500 ঘণ্টা ভিডিও আপলোড হয়। তার মানে 500 × 60 = 30,000 video-hours/day। raw source-এর গড় bitrate ~5 Mbps ধরলে, এক ঘণ্টার ভিডিও প্রায় 2.25 GB। তাহলে raw upload volume ≈ 30,000 × 2.25 GB ≈ **67 TB/day raw ingested video**।

**Transcoding fan-out:** প্রতিটি আপলোড হওয়া ভিডিও ~5-6টি rendition-এ transcode হয় (240p থেকে 1080p/4K পর্যন্ত, প্রতিটি এক বা দুটি codec-এ, যেমন H.264 এবং AV1/VP9)। যদি গড় rendition সেট raw সাইজের প্রায় 1.5-2 গুণ হয় (নিচের resolution গুলো অনেক ছোট, কিন্তু 4K এবং একাধিক codec সেই ওজন ফিরিয়ে দেয়), তাহলে আমরা প্রতিদিন প্রায় **100-150 TB encoded output** সংরক্ষণ করছি, যা raw/master কপি রাখার পাশাপাশি ভবিষ্যতের re-encode-এর জন্য দরকার। এক বছরে এটি কয়েক পেটাবাইট হয়ে দাঁড়ায় — এই কারণেই প্ল্যাটফর্মগুলো tiered, lifecycle-managed object storage ব্যবহার করে।

**Read traffic:** ধরা যাক বিশ্বব্যাপী প্রতিদিন ~5 বিলিয়ন ভিডিও view হয়। তার মানে প্রায় 5,000,000,000 / 86,400 ≈ **58,000 views/second গড়ে**, এবং peak traffic-এ (সন্ধ্যায়, viral ঘটনায়) এটি 3-5 গুণ বেড়ে যায়, ফলে peak-এ **~150,000-250,000 concurrent stream starts/second**।

**CDN egress bandwidth:** যদি গড় concurrent স্ট্রিমিং সেশন ~3 Mbps খরচ করে (মোবাইলে 480p এবং ডেস্কটপে 1080p-র মিশ্রণ), এবং peak-এ বিশ্বব্যাপী প্রায় 50-100 মিলিয়ন concurrent viewer থাকে, তাহলে egress bandwidth হয় 50,000,000 × 3 Mbps = **150 Tbps** global peak-এ (Netflix পাবলিকলি peak সময়ে এই রেঞ্জের সংখ্যা উল্লেখ করেছে)। এমনকি 1 মিলিয়ন concurrent stream-কে 3 Mbps হারে সেবা দেওয়া একটি মাঝারি মাপের প্ল্যাটফর্মেরও **3 Tbps** egress প্রয়োজন — কোনো একক data center এটি প্রদান করতে পারে না; এটি অবশ্যই বিশ্বজুড়ে ছড়িয়ে থাকা edge PoP-সহ একটি CDN থেকে আসতে হবে।

এই সংখ্যাগুলোই পরবর্তী প্রতিটি architectural সিদ্ধান্তকে যুক্তিযুক্ত করে: আমাদের lifecycle tier-সহ object storage, একটি asynchronous fan-out transcoding pipeline, এবং একটি CDN-first delivery মডেল দরকার — এই স্কেলে আমরা সরাসরি origin server থেকে ভিডিও সার্ভ করতে পারি না।

### ধাপ ৩: High-Level Design

উচ্চ স্তরে, pipeline-টি দেখতে এরকম:

**Upload Service → Raw Storage → Transcoding Pipeline (queue-driven, distributed workers) → Encoded Storage / Object Store → Metadata Service → CDN → Client Adaptive Bitrate Player**

ধাপে ধাপে দেখা যাক: একজন ক্লায়েন্ট একটি ভিডিও (প্রায়ই chunks-এ, যাতে অস্থির সংযোগেও resumable upload সম্ভব হয়) একটি load balancer-এর পেছনে থাকা **Upload Service**-এ আপলোড করে (দেখুন [Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md))। upload service raw ফাইলটি **raw/master storage**-এ লেখে — S3 বা GCS-এর মতো durable object storage — এবং একটি "upload complete" event প্রকাশ করে।

সেই event একটি **message queue**-তে (Kafka) পৌঁছায়, যা এটিকে **distributed transcoding workers**-এর একটি pool-এ fan out করে। প্রতিটি worker raw ফাইলটি নেয়, এটিকে একটি encoder-এর মাধ্যমে চালায় (যেমন, FFmpeg-ভিত্তিক pipeline), এবং একাধিক resolution/bitrate rendition তৈরি করে, যা adaptive streaming-এর জন্য ছোট ছোট segment-এ chunk করা থাকে। Encoded output গুলো **encoded storage**-এ লেখা হয় — আবারও একটি object store, কিন্তু এবার একটি CDN origin-এর সাথে যুক্ত।

Transcoding শেষ হলে, **Metadata Service** (একটি sharded database দ্বারা backed) আপডেট হয়: ভিডিওর status "processing" থেকে "ready"-তে পরিবর্তিত হয়, প্রতিটি rendition-এর manifest এবং segment URL, thumbnail, duration ইত্যাদির পয়েন্টারসহ।

প্লেব্যাকের জন্য, **client player** প্রথমে metadata/API layer থেকে ভিডিওর manifest (একটি HLS `.m3u8` বা DASH `.mpd` ফাইল) অনুরোধ করে, এরপর সরাসরি নিকটতম **CDN edge node** থেকে segment স্ট্রিম করে, যেটি হয় সেই segment গুলো cache-এ রেখেছে অথবা cache miss হলে origin (encoded storage) থেকে সেগুলো টেনে আনে। Player ক্রমাগত bandwidth পরিমাপ করে এবং segment-বাই-segment ভিত্তিতে rendition পরিবর্তন করে — এটিই adaptive bitrate streaming।

এই ডিজাইন উদ্বেগগুলো (concerns) পরিষ্কারভাবে আলাদা করে: upload এবং transcoding হলো write-path, asynchronous, এবং horizontally scalable; প্লেব্যাক হলো read-path, CDN-প্রধান, এবং প্রায় তাৎক্ষণিক হতে হবে।

### ধাপ ৪: মূল উপাদানগুলোর গভীর বিশ্লেষণ (Deep Dive)

**১. Asynchronous Transcoding Pipeline (Message Queues, Module 5)।** আমরা কখনোই আপলোডের সময় synchronously transcode করি না — এটি upload সংযোগকে মিনিটের পর মিনিট আটকে রাখবে এবং দুটি সম্পূর্ণ ভিন্ন scaling প্রয়োজনীয়তাকে একসাথে coupled করে ফেলবে। এর পরিবর্তে, upload service শুধু raw ফাইল লেখে এবং একটি Kafka topic-এ একটি মেসেজ ফেলে দেয় ("videoId: X, ready to transcode")। stateless transcoding workers-এর একটি pool সেই topic-এর partition গুলো থেকে consume করে, যা আমাদের upload traffic থেকে স্বাধীনভাবে worker স্কেল করতে এবং ক্লায়েন্টকে প্রভাবিত না করে ব্যর্থ job retry করতে দেয়। এটি **batch vs. stream processing trade-offs**-এর একটি textbook উদাহরণ: একটি একক ভিডিও transcode করা মূলত একটি batch-স্টাইল, CPU-bound কাজ (আমরা পুরো ফাইল, বা তার বড় অংশ, একসাথে প্রসেস করি), অন্যদিকে view counter ও analytics near-real-time আপডেট করা একটি streaming pipeline-এর জন্য বেশি উপযুক্ত। আমরা উভয়ের জন্যই Kafka-কে backbone হিসেবে ব্যবহার করি, কিন্তু ভিন্ন ভিন্ন consumer pattern সহ — transcoding-এর জন্য batch-স্টাইল consumer group, এবং view metrics-এর জন্য streaming aggregation (যেমন, windowed count)।

**২. CDN এবং Edge Caching (Module 4)।** একবার একটি ভিডিও encode হয়ে গেলে, ভৌগোলিক অবস্থান নির্বিশেষে আসল বাইটগুলো ব্যবহারকারীদের কাছে কম latency-তে পৌঁছাতে হবে। আমরা encoded segment গুলো ব্যবহারকারীদের কাছাকাছি edge Points of Presence (PoPs) সহ একটি CDN-এ পুশ করি। জনপ্রিয় ভিডিওগুলো প্রায় প্রতিটি edge node-এ cache করা থাকে (hot content), তাই প্লেব্যাক অনুরোধ কখনো origin-এ পৌঁছায় না। এখানেই **caching strategy** গুরুত্বপূর্ণ: একটি cache-aside pattern সাধারণ, যেখানে CDN edge প্রথম miss-এ origin object storage থেকে fetch করে এবং তারপর একটি TTL-এর জন্য cached কপি সার্ভ করে, invalidation সাধারণত active purge-এর পরিবর্তে content versioning দ্বারা চালিত হয় (যেহেতু encode হওয়ার পর ভিডিও segment গুলো immutable)।

**৩. Adaptive Bitrate Streaming (HLS/DASH)।** প্রতিটি rendition ছোট ছোট segment-এ (প্রতিটি 2-10 সেকেন্ড) chunk করা হয়। player সব উপলব্ধ rendition এবং তাদের segment URL তালিকাভুক্ত করে এমন একটি manifest ডাউনলোড করে, একটি রক্ষণশীল (conservative) bitrate দিয়ে শুরু করে, তারপর throughput পরিমাপ করে এবং segment-বাই-segment ভিত্তিতে quality বাড়ায় বা কমায়। এই কারণেই startup latency কম হয় — player কে প্লেব্যাক শুরু করতে শুধু কম/মাঝারি bitrate-এর প্রথম segment দরকার হয়, পুরো ফাইল নয়।

**৪. Metadata এবং View Counter-এর জন্য Database Sharding (Module 3)।** কোটি কোটি ভিডিও এবং view event থাকায়, একটি একক database metadata টেবিল ধরে রাখতে বা view-count increment-এর write rate সহ্য করতে পারে না। আমরা `videoId` দিয়ে metadata store shard করি (hash-based sharding load সমানভাবে ছড়িয়ে দেয় এবং viral clustering থেকে hot shard এড়ায়)। View counter একটি বিশেষ ক্ষেত্র: প্রতিটি view-তে primary metadata shard-এর একটি row increment করার পরিবর্তে (যা জনপ্রিয় ভিডিওতে বিশাল write contention তৈরি করবে), আমরা streaming pipeline-এর মাধ্যমে view event গুলো batch/aggregate করি এবং periodically aggregated count sharded store-এ flush করি — কয়েক সেকেন্ড/মিনিট counter staleness-এর বিনিময়ে অনেক গুণ কম write load পাওয়া যায়।

### ধাপ ৫: Bottleneck ও Trade-off

- **সব rendition আগে থেকে তৈরি করার তুলনায় Transcoding cost/latency।** প্রতিটি ভিডিওকে সব rendition ও codec-এ তাৎক্ষণিকভাবে transcode করা compute-এ ব্যয়বহুল এবং availability বিলম্বিত করে। একটি বিকল্প হলো শুধু সবচেয়ে সাধারণ rendition গুলো (যেমন, 360p/720p) আগেভাগে transcode করা এবং বিরল গুলো (4K, পুরনো codec) প্রথম অনুরোধের সময় lazily তৈরি করে পরে cache করা। এটি কিছুটা ধীর প্রথম-প্লে-এর বিনিময়ে এমন ভিডিওতে অপচয়িত compute কমায় যা কেউ 4K-তে দেখে না।
- **Hot বনাম long-tail কনটেন্ট caching এবং storage cost।** ভিডিওগুলোর একটি ছোট অংশ (hot/viral কনটেন্ট) বেশিরভাগ view-এর জন্য দায়ী এবং প্রায় প্রতিটি CDN edge-এ থাকা উচিত। long tail — মাত্র কয়েকটি view সহ ভিডিও — ব্যাপক caching-কে যুক্তিসঙ্গত করে না এবং সস্তা, ঠান্ডা (colder) storage class-এ tier করা যেতে পারে, বিরল access-এ বেশি latency মেনে নিয়ে।
- **নতুন viral ভিডিওতে thundering herd।** হঠাৎ viral হওয়া একটি ভিডিও origin storage-কে overwhelm করতে পারে যদি প্রতিটি edge node একসাথে cache miss করে একই segment অনুরোধ করে। প্রতিকারের মধ্যে আছে edge-এ request coalescing (concurrent miss গুলোকে একটি origin fetch-এ একত্র করা), early view-velocity সংকেত trending বোঝালে CDN cache pre-warm করা, এবং origin-shield layer যা fan-in-কে durable storage-এ পৌঁছানোর আগেই শুষে নেয়।
- **Codec/format trade-off।** AV1 বা VP9-এর মতো নতুন codec H.264-এর চেয়ে যথেষ্ট ভালো compression দেয়, storage ও egress cost কমায়, কিন্তু encode করতে বেশি CPU খরচ হয় এবং পুরনো ডিভাইসে সার্বজনীন hardware decode সাপোর্ট কম থাকে। বেশিরভাগ প্ল্যাটফর্ম ব্যাপক compatibility-র জন্য H.264-এ encode করে, পাশাপাশি নতুন ডিভাইস সাপোর্ট এবং দীর্ঘমেয়াদী bandwidth cost কমাতে AV1/VP9 ব্যবহার করে।

### সংক্ষিপ্তসার (Recap)

আমরা upload, transcoding, এবং adaptive playback নিয়ে requirements স্পষ্ট করেছি; সিস্টেমকে দৈনিক শত শত টেরাবাইট storage বৃদ্ধি এবং peak-এ কয়েক দশ থেকে শত টেরাবিট-প্রতি-সেকেন্ড CDN egress-এর হিসেবে আকার দিয়েছি; একটি pipeline ডিজাইন করেছি যা message queue-এর মাধ্যমে upload-কে transcoding থেকে decouple করে; এবং adaptive bitrate streaming ব্যবহার করে delivery-কে CDN-এ ঠেলে দিয়েছি, একটি sharded metadata store দ্বারা backed। আমরা transcoding cost, hot বনাম long-tail কনটেন্ট caching, viral thundering herd, এবং codec choice সংক্রান্ত বাস্তব-জগতের trade-off দিয়ে শেষ করেছি।

### পরবর্তী কী (What's Next)

এই capstone storage, caching, CDN, messaging, এবং sharding-কে একটি সিস্টেমে একত্রিত করেছে। এখান থেকে, কোনো নির্দিষ্ট building block নিয়ে গভীরে যেতে লিঙ্ক করা যেকোনো prerequisite মডিউল পুনরায় দেখুন, অথবা দেখুন কীভাবে live streaming (এই সমস্যার একটি মৌলিকভাবে ভিন্ন, latency-sensitive variant) এই trade-off গুলো পরিবর্তন করে।

## মূল শিক্ষণীয় বিষয় (Key Takeaways)

- write path (upload → async transcoding)-কে read path (CDN-driven playback) থেকে আলাদা করুন — তাদের scaling এবং latency প্রয়োজনীয়তা সম্পূর্ণ ভিন্ন।
- Message queue upload-কে transcoding থেকে decouple করে, স্বাধীন scaling এবং retry সক্ষম করে; transcoding একটি batch-স্টাইল workload, যেখানে view-count aggregation streaming-এর জন্য বেশি উপযুক্ত।
- এই স্কেলে CDN এবং edge caching বাধ্যতামূলক — কোনো origin fleet সরাসরি terabits-per-second egress সার্ভ করতে পারে না।
- ছোট, immutable segment-সহ Adaptive bitrate streaming (HLS/DASH) দ্রুত startup এবং নিরবচ্ছিন্ন quality switching সক্ষম করে।
- Metadata shard করুন এবং view counter-এর মতো উচ্চ-ফ্রিকোয়েন্সি write batch/aggregate করুন hot-shard contention এড়াতে।
- বাস্তব-জগতের trade-off গুলো (pre-transcode বনাম on-demand, hot বনাম long-tail caching, viral thundering herd, codec choice) একটি কার্যকর ডিজাইনকে একটি স্কেলযোগ্য ডিজাইন থেকে আলাদা করে।
</content>
