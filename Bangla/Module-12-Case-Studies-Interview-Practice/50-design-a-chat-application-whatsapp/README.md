# একটি Chat Application ডিজাইন করা (WhatsApp-এর মতো)

**কঠিনতার মাত্রা:** Advanced (Capstone)
**আনুমানিক দৈর্ঘ্য:** 25–30 মিনিট
**পূর্বশর্ত:**
- [WebSockets, Long Polling, and SSE](../../Module-02-Networking-and-Communication/10-websockets-long-polling-and-sse/README.md)
- [Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md)
- [CAP Theorem and PACELC](../../Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md)
- [Message Queues: Kafka vs RabbitMQ](../../Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/README.md)
- [Publish-Subscribe Pattern](../../Module-05-Messaging-and-Asynchronous-Systems/21-publish-subscribe-pattern/README.md)
- [Data Consistency Models and Idempotency](../../Module-06-Distributed-Systems-Concepts/29-data-consistency-models-and-idempotency/README.md)
- [Distributed Caching: Redis and Memcached](../../Module-04-Caching-and-Content-Delivery/19-distributed-caching-redis-and-memcached/README.md)

## শেখার উদ্দেশ্য (Learning Objectives)

- "design a chat application"-এর জন্য একটি সম্পূর্ণ mock interview শুরু থেকে শেষ পর্যন্ত সম্পন্ন করা: requirements, estimation, high-level design, deep dive, trade-offs।
- WhatsApp-এর মতো স্কেলে কাজ করা একটি chat system-এর জন্য capacity (messages/sec, storage, concurrent connections, এবং gateway server সংখ্যা) হিসাব করা।
- WebSocket connection gateways, একটি message queue / pub-sub layer, এবং sharded message storage কীভাবে একসাথে কাজ করে messages route, deliver এবং reliably persist করার জন্য, তা ব্যাখ্যা করা।
- একটি chat system যে মূল trade-offs গুলো করে তা যুক্তিসহ ব্যাখ্যা করা: strict consistency-র বদলে availability, distributed transactions-এর বদলে at-least-once delivery এবং idempotency, এবং message history-র জন্য relational storage-এর বদলে NoSQL wide-column storage।
- ordering, fan-out, multi-device sync, এবং offline delivery নিয়ে interviewer-এর সাধারণ follow-up প্রশ্নগুলো আগে থেকে অনুমান করে সঠিকভাবে উত্তর দেওয়া।

## স্ক্রিপ্ট (Script)

### Hook / Intro

"Design WhatsApp" বা "design a chat application" system design interview-এর সবচেয়ে common capstone প্রশ্নগুলোর একটি, ঠিক এই কারণে যে এটি আপনাকে আগের প্রায় সব module-এর জিনিস একটি সুসংগত system-এ একত্র করতে বাধ্য করে। আপনার real-time bidirectional transport দরকার (WebSockets, Module 2), memory শেয়ার করে না এমন servers-এর মধ্যে messages সরানো দরকার (message queues এবং pub/sub, Module 5), স্কেলে ছোট ছোট writes-এর একটি firehose store করা দরকার (sharding, Module 3), network partition হলে কী ঘটবে তা বিবেচনা করা দরকার (CAP theorem, Module 3), এবং একটি flaky mobile connection যেন duplicate বা lost messages তৈরি না করে তা নিশ্চিত করা দরকার (idempotency, Module 6)। এই ভিডিওতে আমরা এটিকে একটি সম্পূর্ণ mock interview হিসেবে চালাব: requirements স্পষ্ট করা, real numbers দিয়ে system-এর size নির্ধারণ করা, high-level architecture স্কেচ করা, দুই-তিনটি component-এ deep-dive করা যা আসলেই গুরুত্বপূর্ণ, এবং তারপর trade-offs নিয়ে আলোচনা করা — ঠিক যে flow আপনি একটি বাস্তব interview-তে ব্যবহার করবেন।

### Step 1: Requirements স্পষ্ট করা

একটি box আঁকার আগে, উচ্চস্বরে scope স্পষ্ট করুন। এটি interviewer-কে সংকেত দেয় যে আপনি সরাসরি solutions-এ ঝাঁপিয়ে পড়েন না।

**Functional requirements:**
- দুই ব্যবহারকারীর মধ্যে one-to-one messaging।
- Group messaging (প্রতি group-এ কয়েকশ member পর্যন্ত)।
- Delivery receipts (sent → delivered → read, চিরাচরিত single/double/blue check marks)।
- Online presence ("last seen" / online-now indicator)।
- Offline message delivery — recipient-এর device আবার online হলে messages অবশ্যই deliver হতে হবে, হারিয়ে যাওয়া চলবে না।
- Media sharing (images, video, voice notes, documents)।

**Non-functional requirements:**
- Low latency: দুই ব্যবহারকারীই online থাকলে messages এক সেকেন্ডেরও অনেক কম সময়ে পৌঁছানো উচিত।
- High availability: partial failures-এর সময়ও system-এর messages accept এবং forward করা চালিয়ে যাওয়া উচিত — chat হলো এমন একটি product যেখানে ব্যবহারকারীরা একটি single dropped message-ও লক্ষ্য করে।
- At-least-once delivery, client এবং server একসাথে কাজ করে idempotency keys-এর মাধ্যমে approximate exactly-once **effective** delivery অর্জন করবে (ব্যবহারকারীকে duplicate bubbles দেখানো হবে না)।
- Durability: একবার server একটি message acknowledge করলে, তা হারানো চলবে না, এমনকি recipient কয়েক সপ্তাহ offline থাকলেও।

স্পষ্টভাবে বলুন যে আপনি end-to-end encryption key management এবং voice/video calling-কে আলাদা sub-system হিসেবে deprioritize করছেন, এবং trade-offs-এ সংক্ষেপে সেগুলোর উল্লেখ করবেন — এতে interview কেন্দ্রীভূত থাকে।

### Step 2: Capacity Estimation

চলুন whiteboard-এ যেভাবে করতেন, সেভাবে এতে real numbers বসাই।

- **Daily Active Users (DAU):** 500 মিলিয়ন।
- **প্রতি ব্যবহারকারীর দৈনিক গড় messages:** 40।
- **মোট messages/দিন:** 500M × 40 = **20 বিলিয়ন messages/দিন**।
- **গড় messages/sec:** 20,000,000,000 / 86,400 ≈ **230,000 messages/sec**।
- **Peak messages/sec:** chat traffic সমান নয় — এটি বিভিন্ন time zone জুড়ে সন্ধ্যার সময় এবং events-এর সময় (New Year's Eve-টি চিরাচরিত WhatsApp উদাহরণ) spike করে। 3× peak-to-average ratio ব্যবহার করে: ≈ **peak-এ 700,000 messages/sec**।
- **প্রতি message-এ storage:** metadata (message ID, sender ID, conversation ID, timestamp, delivery status, ছোট text body) গড়ে প্রায় **100 bytes**। তাতে শুধু metadata-র জন্যই 20B × 100 bytes ≈ **2 TB/দিন**, বা replication ছাড়া প্রায় 700 TB/বছর হয় — durability-র জন্য 3× replication সহ, ধরে নিন message metadata store-এর জন্য এটি **~2 PB/বছর**। Media inline store করা হয় **না**; এটি একটি আলাদা blob store-এ upload করা হয় (S3-এর মতো object storage), message row-তে শুধু একটি URL/reference থাকে, কারণ অল্প শতাংশ messages-এ multi-megabyte attachment থাকে এবং সেটিকে hot metadata table-এর সাথে মেশালে write latency নষ্ট হয়ে যেত।
- **Concurrent WebSocket connections:** 500M DAU-র সবাই একই সময়ে online থাকে না। Peak-এ ~30% concurrency ধরে নিন (messaging apps-এর জন্য একটি common rule of thumb): **150 মিলিয়ন concurrent persistent connections**।
- **প্রয়োজনীয় connection gateway servers:** প্রতি gateway server-এ প্রায় 50,000 concurrent connections ধরলে (idle WebSocket connections ধরে রাখা একটি tuned event-loop server-এর জন্য একটি বাস্তবসম্মত ceiling), তা 150,000,000 / 50,000 = **3,000 gateway servers**, regions জুড়ে redundancy এবং headroom সহ ~3,500–4,000-এ raun up করা হয়।

এই numbers-গুলোই পরবর্তী প্রতিটি design decision চালিত করে: এগুলো আমাদের বলে যে storage-এর জন্য horizontal sharding দরকার, stateful connection gateways-এর একটি fleet দরকার (একটি single load-balanced stateless pool নয়), এবং ingestion থেকে delivery-কে decouple করার জন্য একটি asynchronous queue দরকার।

### Step 3: High-Level Design

request path বাম থেকে ডানে স্কেচ করুন:

```
Client (mobile/web)
  → Load Balancer (L4, sticky-ish routing)
    → Connection Gateway servers (WebSocket termination)
      → Chat/Message Service (business logic, validation, idempotency check)
        → Message Queue (Kafka-based pub/sub)
          → Message Store (sharded, wide-column DB) — persists every message
          → Presence Service — tracks which gateway server each user is connected to
          → Push Notification Service — wakes up offline devices via APNs/FCM
```

flow-টি ধাপে ধাপে দেখুন: একটি client একটি **Connection Gateway** server-এর সাথে একটি persistent WebSocket connection খোলে (একটি load balancer-এর মাধ্যমে যেটি একটি gateway বেছে নেয় এবং তারপর থেকে, সেই TCP/WebSocket connection সেই একটি server-এর সাথেই pinned থাকে — এই "stickiness"-টি আমরা Step 5-এ আবার দেখব)। ব্যবহারকারী একটি message পাঠালে, এটি সেই WebSocket-এর মাধ্যমে gateway-তে যায়, যেটি একে **Chat Service**-এ forward করে। Chat Service একটি message ID assign করে, **Presence Service** চেক করে জানার জন্য যে recipient বর্তমানে কোন gateway server-এর সাথে connected (যদি থাকে), message-টি **Message Store**-এ persist করে, এবং **Message Queue**-তে একটি event publish করে। Recipient online থাকলে, message-টি — সম্ভবত queue-এর pub/sub layer-এর মাধ্যমে — নির্দিষ্ট gateway server-এ route করা হয় যেটি recipient-এর connection ধরে রেখেছে, এবং real time-এ সেই WebSocket-এর মাধ্যমে push করা হয়। Recipient offline থাকলে, **Push Notification Service** APNs (iOS) বা FCM (Android)-এর মাধ্যমে একটি silent/data push পাঠায় যাতে OS app-টিকে জাগিয়ে তোলে, যা তখন reconnect করে এবং Message Store থেকে miss হওয়া messages-গুলো pull করে।

### Step 4: মূল Components-এ Deep Dive

এখানেই আপনি design-টিকে fundamentals-এর সাথে সংযুক্ত করেন — ঠিক যা একজন interviewer শুনতে চান।

**4a. WebSockets-এর মাধ্যমে real-time transport (Module 2)।** একটি chat app-এর full-duplex, low-latency communication দরকার, এবং server-কে client-কে আগে জিজ্ঞেস না করেই push করতে সক্ষম হতে হবে — যা plain HTTP polling-কে বাদ দেয়। আমরা WebSockets ব্যবহার করি, প্রতি client session-এ একবার establish করা হয় এবং app foreground-এ থাকা পর্যন্ত খোলা রাখা হয় (dead connections detect করতে এবং gateway capacity মুক্ত করতে heartbeats/pings সহ)। এটি ঠিক সেই trade-off যা WebSockets/long-polling/SSE module-এ আলোচনা করা হয়েছে: WebSockets আমাদের bidirectional push দেয়, তার বিনিময়ে gateway server-কে memory-তে লক্ষ লক্ষ stateful, বেশিরভাগ-idle connections ধরে রাখতে হয় — তাই উপরের 50K-connections-per-server sizing।

**4b. একটি message queue / pub-sub দিয়ে gateway servers-এর মধ্যে routing (Module 5)।** যেহেতু connections নির্দিষ্ট gateway servers-এ pinned থাকে, sender এবং recipient খুব সম্ভবত *ভিন্ন* physical gateway servers-এর সাথে connected থাকে। Chat Service স্কেলে সরাসরি recipient-এর gateway-কে "call" করতে পারে না — পরিবর্তে এটি outgoing message-কে recipient-এর gateway server ID দিয়ে keyed একটি topic-এ publish করে (Kafka, message-queues module-এ বর্ণিত), অথবা একটি pub/sub fan-out ব্যবহার করে (Redis Pub/Sub বা প্রতি gateway-র জন্য একটি Kafka consumer group) যেখানে প্রতিটি gateway তার সাথে বর্তমানে connected user ID-গুলোর জন্য একটি channel subscribe করে। এটি "কে পাঠিয়েছে" থেকে "কে live connection ধরে রেখেছে"-কে decouple করে, এবং এটি আমাদের একটি durable buffer-ও দেয়: একটি gateway server delivery-র মাঝামাঝি crash করলে, message-টি হারিয়ে যায় না — এটি এখনও queue/store-এ থাকে এবং redeliver করা হয়।

**4c. Message history-র জন্য sharded storage (Module 3)।** প্রতি বছর ~2 PB message metadata নিয়ে, একটি single database instance কোনো বিকল্পই নয়। আমরা Message Store-কে **conversation ID** দিয়ে shard করি (1:1 chats-এর জন্য, দুই user ID-র একটি deterministic hash; groups-এর জন্য, group ID) যাতে একটি নির্দিষ্ট conversation-এর সব messages একই shard-এ পড়ে এবং একটি single range query দিয়ে order অনুযায়ী fetch করা যায় — এটি ঠিক সেই conversation-locality pattern যা আপনি অন্য যেকোনো append-heavy, read-by-partition-key workload sharding করার সময় ব্যবহার করবেন। একটি wide-column store (Cassandra/HBase-style, বা DynamoDB) ভালোভাবে ফিট করে কারণ writes append-only এবং reads প্রায় সবসময়ই "এই conversation-এর জন্য শেষ N messages আমাকে দাও" ধরনের।

**4d. Idempotency এবং delivery guarantees (Module 6)।** Mobile networks ক্রমাগত drop এবং retry করে, তাই client-to-server hop স্বভাবতই at-least-once: client সময়মতো ACK না পেলে message পুনরায় পাঠাবে। duplicate bubbles দেখানো এড়াতে, client পাঠানোর আগে একটি client-side message ID (UUID) তৈরি করে; Chat Service persist করার আগে সেই ID (একটি idempotency key)-তে deduplicate করে। Delivery status (sent/delivered/read) প্রতি message-এ একটি ছোট state machine হিসেবে track করা হয়, ACKs recipient-এর device থেকে একই gateway → queue → sender-এর gateway path দিয়ে ফিরে আসে।

**4e. CAP trade-offs (Module 3)।** একটি network partition-এর সময়, আমরা স্পষ্টভাবে **strict consistency-র বদলে availability** বেছে নিই — একজন ব্যবহারকারীর জন্য একটি message পাঠানো যা কয়েক সেকেন্ড দেরিতে বা devices জুড়ে সামান্য ভুল order-এ পৌঁছায়, তা send button একেবারে fail করার চেয়ে অনেক ভালো। আমরা read-receipt propagation এবং presence status-এর মতো জিনিসগুলোতে eventual consistency মেনে নিই, একইসাথে message durability-কে (একটি persisted message কখনো silently drop না করা) non-negotiable হিসেবে বিবেচনা করি।

### Step 5: Bottlenecks এবং Trade-offs

- **Connection stickiness এবং gateways স্কেল করা।** যেহেতু প্রতিটি WebSocket একটি gateway server-এ pinned থাকে, scale out করার মানে হলো gateway servers যোগ করা এবং নতুন connections-গুলোকে সেগুলোর মধ্যে rebalance করা — কিন্তু existing connections-গুলো শুধু "move" করা যায় না। একটি gateway restart/deploy-কে gracefully connections drain করতে হয় এবং clients-দের অন্য কোথাও reconnect করতে দিতে হয়, এবং connection move হওয়ার সাথে সাথেই Presence Service update করতে হয়।
- **Message ordering।** একটি single 1:1 conversation-এর মধ্যে, যদি সেই conversation-এর সব messages একই shard/partition দিয়ে যায় তাহলে ordering সোজা। বহু concurrent senders সহ group chats-এ, আমরা সাধারণত global total order-এর জন্য খরচ করার বদলে, প্রতি conversation-এ per-message timestamps এবং sequence numbers সহ "causal-ish" ordering মেনে নিই।
- **Group chat fan-out।** 500 জনের একটি group-এ একটি message মানে 500টি আলাদা deliveries। স্কেলে, এটি queue-এর মাধ্যমে asynchronously করা হয় (active groups-এর জন্য fan-out-on-write, কখনো কখনো খুব বড় বা inactive groups-এর জন্য fan-out-on-read) যাতে একজন slow recipient বাকি 499 জনকে কখনো block না করে।
- **Storage growth এবং archival।** metadata-র জন্য বছরে ~2 PB সহ media volume আরও অনেক বড়, hot storage সবকিছু চিরকাল ধরে রাখতে পারে না। সাম্প্রতিক messages fast, sharded hot storage-এ থাকে; পুরনো conversation history typical time-based tiering strategies অনুযায়ী সস্তা cold/archival storage-এ (একটি index সহ object storage) সরানো হয়।
- **Offline devices-এর জন্য push notification delivery।** APNs/FCM delivery guaranteed বা instant নয়, এবং payload size সীমিত, তাই pushes শুধু app-টিকে জাগানোর মতো যথেষ্ট তথ্য বহন করে, যেটি তখন Message Store থেকে প্রকৃত messages pull করে — push path একটি "doorbell", message transport নিজেই নয়।

### Recap

আমরা functional requirements (1:1 এবং group messaging, receipts, presence, offline delivery, media) এবং non-functional requirements (low latency, high availability, idempotency-সহ at-least-once, durability) স্পষ্ট করেছি; system-টিকে 20B messages/দিন (~230K/sec গড়, ~700K/sec peak), বছরে ~2 PB message metadata, এবং 150M concurrent connections-এর জন্য ~3,000+ gateway servers প্রয়োজন এভাবে size করেছি; client থেকে Connection Gateway, Chat Service, Message Queue, sharded Message Store, Presence Service, এবং Push Notification Service পর্যন্ত high-level path walk করেছি; WebSockets, gateways-এর মধ্যে pub/sub routing, conversation-ID sharding, এবং idempotent at-least-once delivery-তে deep-dive করেছি; এবং stickiness, ordering, fan-out, storage tiering, এবং push delivery নিয়ে trade-offs দিয়ে শেষ করেছি।

### এরপর কী

এই case study একটি capstone — যদি কোনো অংশ অপরিচিত মনে হয়, উপরের linked prerequisite modules-এ (WebSockets, sharding, CAP theorem, message queues, pub/sub, idempotency, এবং caching) ফিরে যান, আমরা এখানে যে component-টির শুধু সারাংশ দেওয়ার সময় পেয়েছি তার পেছনের deep-dive theory-র জন্য। Interview-practice দৃষ্টিকোণ থেকে, এই script-এর সাথে আপনার version মিলিয়ে দেখার আগে একটি blank page-এ শুরু থেকে এই design পুনরায় derive করার চেষ্টা করুন, তারপর এই folder-এর quiz-এ চলে যান নিজেকে পরীক্ষা করার জন্য একজন প্রকৃত interviewer যে ধরনের follow-up প্রশ্ন ছুঁড়ে দেবে তার বিরুদ্ধে।

## মূল বিষয়গুলো (Key Takeaways)

- Chat systems একটি capstone problem কারণ এগুলোর জন্য real-time transport (WebSockets), asynchronous routing (queues/pub-sub), sharded persistence, এবং idempotent delivery — সবগুলো একসাথে কাজ করা প্রয়োজন।
- Design করার আগে সবসময় real numbers দিয়ে capacity estimate করুন — 500M DAU × 40 messages/day থেকে 20B messages/day হয়, ~230K/sec গড় এবং ~700K/sec peak, যা সরাসরি horizontal sharding এবং একটি dedicated stateful gateway fleet-কে justify করে।
- Persistent connections নির্দিষ্ট gateway servers-এ pinned থাকে, তাই sender-এর gateway থেকে recipient-এর gateway-তে একটি message route করার জন্য একটি queue বা pub/sub layer প্রয়োজন।
- Message history-কে conversation ID দিয়ে shard করুন যাতে একটি conversation-এর messages fast, ordered range reads-এর জন্য একসাথে থাকে।
- Client-generated message IDs এবং server-side deduplication স্বভাবতই at-least-once একটি network-কে ব্যবহারকারীর জন্য কার্যত exactly-once একটি experience-এ রূপান্তরিত করে।
- Partitions-এর সময় strict consistency-র বদলে availability বেছে নিন; perfect real-time ordering নয়, message durability-কে non-negotiable guarantee হিসেবে বিবেচনা করুন।
