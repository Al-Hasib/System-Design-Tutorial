# একটি Ride-Sharing System ডিজাইন করা (Uber-এর মতো)

**কঠিনতার মাত্রা:** Advanced (Capstone)। **প্রয়োজনীয় পূর্বজ্ঞান:** [WebSockets, Long Polling, and SSE](../../Module-02-Networking-and-Communication/10-websockets-long-polling-and-sse/README.md), [Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md), [CAP Theorem and PACELC](../../Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md), [Publish-Subscribe Pattern](../../Module-05-Messaging-and-Asynchronous-Systems/21-publish-subscribe-pattern/README.md), [Consistent Hashing Explained](../../Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/README.md), [Distributed Transactions: 2PC and Saga](../../Module-06-Distributed-Systems-Concepts/28-distributed-transactions-2pc-and-saga/README.md), [Data Consistency Models and Idempotency](../../Module-06-Distributed-Systems-Concepts/29-data-consistency-models-and-idempotency/README.md)

## শেখার লক্ষ্য (Learning Objectives)

- একটি ride-sharing প্ল্যাটফর্মকে পরস্পর সহযোগিতাকারী কিছু service-এর সমষ্টি হিসেবে দেখা: location, matching, trip lifecycle, এবং payment।
- ক্রমাগত location স্ট্রিমসহ একটি geospatially-distributed, real-time system-এর জন্য capacity estimation করা।
- বড় স্কেলে location data শার্ড ও কোয়েরি করতে geohashing/quadtree-কে consistent hashing-এর সাথে একত্রে প্রয়োগ করা।
- একটি multi-step, multi-service trip flow-কে compensating action এবং idempotent payment call সহ একটি Saga হিসেবে মডেল করা।
- location data (availability-favored) এবং payment data (consistency-favored)-এর জন্য CAP/PACELC trade-off আলাদাভাবে বিশ্লেষণ করা।

## স্ক্রিপ্ট (Script)

### সূচনা / ভূমিকা

কল্পনা করুন: রাত ১১টা, একটি কনসার্ট মাত্র শেষ হয়েছে, এবং তিন হাজার মানুষ একই দুই মিনিটের মধ্যে তাদের ride-sharing app খুলছে, সবাই এক চতুর্থাংশ মাইলের ব্যাসার্ধের মধ্যে দাঁড়িয়ে। System-কে সেকেন্ডের মধ্যে তাদের জন্য একজন driver খুঁজে বের করতে হবে, সেই driver ট্রাফিকের মধ্য দিয়ে যাওয়ার সময় তার সঠিক অবস্থান track করতে হবে, বাড়তি চাহিদা প্রতিফলিত করে একটি fare হিসাব করতে হবে, এবং শেষে একটি কার্ড থেকে টাকা কাটতে হবে — কোনো একটি driver-কে দুইবার বুক না করে, কোনো একজন rider-কে দুইবার চার্জ না করে। এটাই ride-sharing system design সমস্যা, এবং এটি সেরা interview প্রশ্নগুলোর একটি, কারণ এটি আপনাকে geospatial indexing, real-time messaging, distributed transaction, এবং consistency trade-off—সবকিছুকে একটি সুসংগত ডিজাইনে একত্র করতে বাধ্য করে। আজ আমরা শূন্য থেকে Uber বা Lyft-এর মতো একটি system ডিজাইন করব। প্রথমে আসুন প্রয়োজনীয়তাগুলো স্পষ্ট করি।

### ধাপ ১: প্রয়োজনীয়তা স্পষ্ট করা (Clarify Requirements)

বাক্স আঁকার আগে, আমরা interviewer-এর সাথে scope নির্ধারণ করি।

**Functional requirements:**
- একজন rider পিকআপ location এবং destination উল্লেখ করে একটি ride অনুরোধ করতে পারবেন।
- System rider-কে কাছাকাছি একজন available driver-এর সাথে match করবে।
- Rider এবং driver উভয়েই approach এবং trip চলাকালীন একে অপরের live GPS location একটি ম্যাপে দেখতে পাবেন।
- System distance, time এবং demand (surge pricing) এর ভিত্তিতে fare হিসাব করবে।
- System পুরো trip lifecycle পরিচালনা করবে — requested, driver assigned, driver arrived, trip started, trip completed — এবং শেষে payment প্রক্রিয়া করবে।

**Non-functional requirements:**
- **Low-latency matching**: একজন rider-কে দশ সেকেন্ডের নয়, বরং কয়েক সেকেন্ডের মধ্যে একজন driver match পাওয়া উচিত।
- **High availability**: আঞ্চলিক infrastructure সমস্যার সময়েও app ব্যবহারযোগ্য থাকতে হবে — একটি আটকে যাওয়া request সাথে সাথেই বিশ্বাস ও রাজস্ব হারায়।
- **Geospatial scale**: বিশ্বব্যাপী লক্ষ লক্ষ চলমান driver জুড়ে system-কে দক্ষতার সাথে "who is near me" প্রশ্নের উত্তর দিতে হবে।
- **যেখানে গুরুত্বপূর্ণ সেখানে consistency**: trip state (একজন driver-কে দুইবার বুক করা যাবে না) এবং payment (একজন rider-কে দুইবার চার্জ করা যাবে না, এবং একটি সম্পন্ন trip-এর জন্য অবশেষে টাকা পরিশোধ করতে হবে) এর জন্য strong consistency guarantee প্রয়োজন, যদিও location ping সামান্য staleness সহ্য করতে পারে।

আমরা স্পষ্টভাবে উল্লেখ করি যে আমরা location data-এর জন্য availability এবং trip-state ও payment data-এর জন্য consistency-কে অগ্রাধিকার দিচ্ছি — এই বিভাজনটিই পরবর্তীতে বেশিরভাগ গুরুত্বপূর্ণ সিদ্ধান্ত নিয়ন্ত্রণ করে।

### ধাপ ২: Capacity Estimation

আসুন এটিকে বাস্তব সংখ্যায় প্রতিষ্ঠিত করি। ধরা যাক আমাদের **২ কোটি (20 million) daily active rider** এবং **৫০ লক্ষ (5 million) daily active driver** আছে, যাদের মধ্যে peak hour-এ প্রায় **২০ লক্ষ (2 million) driver একই সাথে active** থাকেন।

**Location updates:** প্রতিটি active driver-এর app প্রতি ৪ সেকেন্ডে একটি GPS ping পাঠায়। তাহলে:

2,000,000 driver / 4 সেকেন্ড ≈ peak-এ **প্রতি সেকেন্ডে 500,000 location update**।

এটিই পুরো system-এর প্রধান write load — ride request-এর চেয়েও অনেক বেশি — এবং এটি আমাদের সাথে সাথে বলে দেয় যে location pipeline-কে একটি traditional ACID database নয়, বরং high-throughput, low-durability-requirement লেখার জন্য তৈরি করতে হবে।

**Ride requests:** ধরা যাক ২ কোটি rider গড়ে দিনে ১.২টি ride নেন, যা মোটামুটি ২৪০ লক্ষ (24 million) rides/day দেয়। ৮৬,৪০০ সেকেন্ডের গড় হিসাবে, তা প্রায় সেকেন্ডে ২৮০টি request, কিন্তু ride demand বার্স্টি — rush hour, কনসার্ট, খারাপ আবহাওয়া — তাই আমরা **প্রতি সেকেন্ডে প্রায় 800-1,000 ride request-এর peak** ধরে পরিকল্পনা করি।

**Trip history storage:** প্রতিটি trip-এর metadata (~1 KB: rider ID, driver ID, timestamp, fare, status) এবং একটি GPS trace থাকে। প্রতি ৪ সেকেন্ডে ping করা একটি ১৫-মিনিটের trip প্রায় ২২৫টি পয়েন্ট তৈরি করে, প্রতিটি ~50 বাইট, বা ~11 KB trace data। তাহলে প্রতি trip-এ মোটামুটি ১২ KB, এবং দিনে ২৪০ লক্ষ trip-এ তা প্রায় **প্রতিদিন 290 GB**, বা বছরে প্রায় **100 TB** — hot/warm tiering strategy সহ cold object storage-এর জন্য খুবই সামলানোর মতো।

এই সংখ্যাগুলো দুটি ডিজাইন সিদ্ধান্তকে যুক্তিসঙ্গত করে: (১) location data-এর জন্য একটি in-memory, horizontally-sharded store দরকার, কোনো relational database নয়, এবং (২) trip ও payment record তুলনামূলকভাবে কম-ভলিউমের এবং শক্তিশালী consistency mechanism বহন করতে পারে।

### ধাপ ৩: High-Level Design

উচ্চ স্তরে, rider এবং driver mobile app থেকে আসা request একটি **load balancer**-এ পৌঁছায়, তারপর একটি **API gateway**-তে যা auth, rate limiting এবং routing পরিচালনা করে। সেখান থেকে:

- একটি **Location Service** সেকেন্ডে 500K driver GPS ping গ্রহণ করে এবং "এই মুহূর্তে প্রতিটি driver কোথায় আছে" এর একটি geospatial index বজায় রাখে।
- একটি **Matching Service** একটি ride request নিয়ে Location Service-কে কাছাকাছি available driver-দের জন্য কোয়েরি করে, candidate-দের score ও rank করে (distance, ETA, driver rating), এবং একজনকে assign করে।
- একটি **Trip Service** trip state machine-এর মালিকানা রাখে (requested → assigned → arrived → in-progress → completed → paid) এবং trip status-এর জন্য source of truth।
- একটি **Payment Service** fare calculation, surge pricing পরিচালনা করে এবং trip সম্পন্ন হলে rider-এর সংরক্ষিত payment method থেকে টাকা কাটে।
- Live position update উভয় app-এ একটি **pub/sub layer** এর মাধ্যমে প্রবাহিত হয় যা **WebSocket** connection-কে ফিড করে, ফলে rider প্রায় real time-এ কোনো polling ছাড়াই ম্যাপে driver icon-কে নড়তে দেখেন।

এর নিচে, আমরা live driver location-এর জন্য একটি geospatially-partitioned key-value store ব্যবহার করব (যেমন geo command সহ Redis, অথবা একটি custom geohash-sharded store), trip ও payment record-এর জন্য একটি relational বা strongly-consistent store, এবং pub/sub fan-out ও trip saga চলাকালীন inter-service coordination-এর জন্য একটি message broker।

### ধাপ ৪: মূল Component-গুলোর গভীর বিশ্লেষণ (Deep Dive)

**Geospatial indexing এবং consistent hashing।** "এই পয়েন্টের 2 km-এর মধ্যে available driver খুঁজুন" প্রশ্নের দ্রুত উত্তর দিতে, লক্ষ লক্ষ driver জুড়ে, আমরা প্রতিটি রেকর্ড scan করতে পারি না। আমরা **geohashing** (বা একটি **quadtree**) ব্যবহার করি lat/long-কে এমন একটি string বা tree path-এ রূপান্তরিত করতে যা কাছাকাছি পয়েন্টগুলোকে একত্রে ক্লাস্টার করে — একটি common geohash prefix মানে শারীরিক নৈকট্য। এটি কোয়েরি সমস্যার সমাধান করে, কিন্তু আমাদের *distribution* সমস্যারও সমাধান দরকার: Chicago শহরের downtown-এর geohash bucket কোন server-এ রাখা হবে? এখানেই Module 6-এর **consistent hashing** কাজে আসে — আমরা geohash prefix hash করি (অথবা সরাসরি hash-ring key হিসেবে ব্যবহার করি) location shard-কে node-এ assign করতে, ফলে একটি location-service node যোগ বা অপসারণ করলে পুরো dataset নয়, বরং শুধু geographic cell-এর একটি ছোট অংশ পুনর্বিন্যস্ত হয়। Geohash আমাদের কোয়েরির জন্য locality দেয়; consistent hashing সেই data ধারণকারী server-এর গুচ্ছ জুড়ে locality-preserving, low-churn distribution দেয়।

**Real-time location broadcast।** একবার একটি trip match হয়ে গেলে, rider এবং driver উভয় app-এরই driver-এর position-এর একটি live feed দরকার। Driver-এর app প্রতিটি GPS ping সেই trip-এর জন্য নির্দিষ্ট একটি **pub/sub** topic-এ publish করে (Module 5-এর publish-subscribe pattern-এ যেমন আলোচিত) — Location Service বা একটি হালকা relay publish করে, এবং সেই trip ID-তে আগ্রহী যেকোনো subscriber তা পায়। প্রতিটি rider এবং driver app তাদের active trip-এর topic-এ subscribe করা একটি gateway-এর সাথে একটি persistent **WebSocket** connection (Module 2) বজায় রাখে, ফলে client-কে প্রতি কয়েক সেকেন্ডে একটি HTTP endpoint poll করার বদলে update তাৎক্ষণিকভাবে push হয়। এই সমন্বয় — fan-out-এর জন্য pub/sub, delivery-এর জন্য WebSocket — "আপনার driver-কে এগিয়ে আসতে দেখা" অভিজ্ঞতাকে জীবন্ত মনে করায়।

**Trip lifecycle একটি Saga হিসেবে।** একটি একক ride একাধিক service স্পর্শ করে: একজন driver সংরক্ষণ করা, উভয় পক্ষের সাথে ride নিশ্চিত করা, trip চালানো, এবং একটি payment চার্জ করা — এবং এগুলো আলাদা datastore-সহ আলাদা service, তাই সবগুলোজুড়ে একটি ক্লাসিক ACID transaction ব্যবহারিক নয়, এবং two-phase commit (Module 6) পুরো trip-এর সময়কাল জুড়ে service-গুলোর মধ্যে lock ধরে রাখবে, যা টেকসই নয়। এর পরিবর্তে আমরা trip-কে একটি **Saga** হিসেবে মডেল করি: local transaction-এর একটি ধারাবাহিকতা, যার প্রতিটির জন্য পরবর্তী কোনো ধাপ ব্যর্থ হলে একটি নির্দিষ্ট **compensating action** থাকে। ধাপ ১: সাময়িকভাবে driver সংরক্ষণ করা (তাকে "pending" হিসেবে চিহ্নিত করা, পুরোপুরি বুক নয়)। ধাপ ২: উভয় app-এর সাথে match নিশ্চিত করা। ধাপ ৩: trip চালানো এবং completed হিসেবে চিহ্নিত করা। ধাপ ৪: payment চার্জ করা। যদি ধাপ ৪-এ payment ব্যর্থ হয়, compensating action হতে পারে একটি backup payment method দিয়ে retry করা, তারপর একটি ইতিমধ্যে ঘটে যাওয়া শারীরিক trip-কে "undo" করার চেষ্টা না করে trip-টিকে collection-এর জন্য flag করা — একটি saga-তে compensation মানে business-level rollback, আক্ষরিক reversal নয়। যদি ধাপ ১-এ driver reservation ব্যর্থ হয় বা timeout হয়, আমরা কেবল driver-কে ছেড়ে দিই এবং matching পুনরায় চালাই — কোনো downstream ধাপ এখনো ঘটেনি, তাই compensate করার মতো কিছু নেই। গুরুত্বপূর্ণভাবে, payment charge call-এ trip ID থেকে উদ্ভূত একটি **idempotency key** থাকে (Module 6-এর consistency/idempotency concept) — যদি একটি network blip-এর কারণে Payment Service একই "charge this trip" request দুইবার পায়, দ্বিতীয় call-টিকে duplicate হিসেবে চিহ্নিত করে মূল ফলাফল ফেরত দেওয়া হয়, rider-কে দুইবার চার্জ করার বদলে।

### ধাপ ৫: Bottleneck ও Trade-off

কয়েকটি টানাপোড়েন স্পষ্টভাবে তুলে ধরার মতো, কারণ একজন ভালো candidate নিখুঁত উত্তর আছে এমন ভান না করে trade-off-এর নাম বলে।

**Matching accuracy বনাম latency।** আমরা একটি বিশাল radius search করতে পারি এবং প্রতিটি candidate driver-কে objectively সেরা match-এর জন্য গভীরভাবে score করতে পারি, কিন্তু এটি rider-এর অনুভব করা latency যোগ করে। বাস্তবে আমরা প্রথমে একটি ছোট radius search করি, driver না পেলেই কেবল সম্প্রসারণ করি, এবং exhaustive optimization-এর বদলে হালকা heuristics (distance, ETA) ব্যবহার করি — এই UX-এর জন্য "যথেষ্ট ভালো, দ্রুত" "optimal, ধীর"-এর চেয়ে ভালো।

**Hot zone এবং surge pricing।** যখন একটি কনসার্ট শেষ হয়, একটি ছোট geohash cell-এ demand স্থানীয় driver সরবরাহের চেয়ে অনেক বেশি বেড়ে যায়। Matching Service-এর এই ভারসাম্যহীনতা শনাক্ত করা উচিত (একটি cell-এ available driver-এর তুলনায় open ride request অনেক বেশি) এবং Payment Service **surge pricing** প্রয়োগ করে — একইসাথে price-এর মাধ্যমে সীমিত সরবরাহ রেশনিং করতে এবং কাছাকাছি driver-দের hot zone-এর দিকে পুনঃস্থাপিত হতে উৎসাহিত করতে।

**Concurrency-এর অধীনে driver availability-এর consistency।** দুটি ride request প্রায় একই মুহূর্তে একই driver-কে match করার চেষ্টা করতে পারে। এটি একটি ক্লাসিক race condition, এবং এই কারণেই Saga-এর প্রথম ধাপটি একটি optimistic assignment না হয়ে একটি conditional, atomic "reserve" অপারেশন (যেমন driver status-এ একটি compare-and-set) — আমাদের সেই একটি flag-এর জন্য strong consistency দরকার, এমনকি এমন একটি system-এও যা অন্যথায় high throughput-এর জন্য তৈরি।

**প্রতিটি subsystem-এ ভিন্নভাবে প্রয়োগ করা CAP trade-off।** Location data উচ্চ-ভলিউমের, ঘনঘন overwrite হয়, এবং staleness সহনশীল — একটি driver pin অর্ধ সেকেন্ড পুরনো হওয়া ক্ষতিকর নয় — তাই আমরা সেখানে **availability এবং partition tolerance** (AP)-কে অগ্রাধিকার দিই, eventual consistency মেনে নিয়ে। Payment এবং trip-state data কম-ভলিউমের কিন্তু কখনো ভুল হওয়া চলবে না — আমরা সেখানে **consistency** (CP)-কে অগ্রাধিকার দিই, মেনে নিয়ে যে একটি node partition সংক্ষিপ্তভাবে একটি write প্রত্যাখ্যান করতে পারে বরং একটি double-charge বা double-booking-এর ঝুঁকি নেওয়ার চেয়ে। এটিই Module 3-এর মূল CAP/PACELC শিক্ষা: সঠিক পছন্দ সর্বজনীন নয়, এটি প্রতিটি subsystem-ভিত্তিক, একটি inconsistency আসলে কী খরচ করবে তার উপর ভিত্তি করে।

### সারসংক্ষেপ (Recap)

আমরা একটি ride-sharing platform ডিজাইন করেছি বিষয়গুলোকে ভাগ করে: geohashing এবং consistent hashing ব্যবহার করে বিশাল-throughput, availability-favored write-এর জন্য তৈরি একটি Location Service; latency এবং match quality-এর মধ্যে ভারসাম্য রক্ষাকারী একটি Matching Service; compensating action সহ lifecycle-কে একটি Saga হিসেবে চালানো একটি Trip Service; এবং idempotency key দ্বারা সুরক্ষিত একটি Payment Service। Real-time tracking pub/sub এবং WebSocket-এর উপর নির্ভর করে। এবং সর্বত্র, আমরা একটি একক সার্বজনীন consistency model-এর বদলে ইচ্ছাকৃত, subsystem-নির্দিষ্ট CAP trade-off করেছি।

### পরবর্তী কী (What's Next)

এই case study geospatial indexing, pub/sub, saga, এবং CAP trade-off-এর উপর ব্যাপকভাবে নির্ভর করেছে — যদি এগুলোর যেকোনোটি নতুন উপাদানের বদলে একটি পুনরালোচনা মনে হয়ে থাকে, তবে সেটাই মূল বিষয়: capstone case study-গুলো আসলে দেখানোর জন্যই আছে যে প্রকৃত interview চাপের অধীনে আগের module-গুলোর পৃথক building block-গুলো কীভাবে একত্রিত হয়। আপনার পরবর্তী mock interview-এর আগে নিজে এই ডিজাইনটি স্কেচ করার চেষ্টা করুন, এবং আপনার fare-calculation ও surge-pricing পদ্ধতিকে আমাদের সাথে তুলনা করুন।

## মূল শিক্ষণীয় বিষয় (Key Takeaways)

- System-কে বিষয় অনুযায়ী ভাগ করুন: Location Service, Matching Service, Trip Service (state machine), এবং Payment Service, প্রতিটির নিজস্ব consistency প্রয়োজনীয়তা সহ।
- Location data হাই-throughput এবং staleness-tolerant — availability (AP)-কে অগ্রাধিকার দিন; trip state এবং payment data সঠিক হতেই হবে — consistency (CP)-কে অগ্রাধিকার দিন।
- Geohashing/quadtree "nearby driver" কোয়েরির জন্য locality দেয়; consistent hashing সেই geospatial data-এর server জুড়ে low-churn distribution দেয়।
- Pub/sub এবং WebSocket একসাথে client-side polling ছাড়াই live location tracking প্রদান করে।
- Multi-service trip flow-কে two-phase commit নয়, বরং compensating action সহ একটি Saga হিসেবে মডেল করুন — এবং double-charging প্রতিরোধ করতে payment call-কে idempotency key দিয়ে সুরক্ষিত করুন।
- একটি interview-এ trade-off-গুলো স্পষ্টভাবে উল্লেখ করুন: matching speed বনাম accuracy, এবং প্রতি-subsystem CAP পছন্দ, কোনো একটি ডিজাইনকে "নিখুঁত" বলে দাবি করার বদলে।
