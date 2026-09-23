# একটি Distributed File Storage System ডিজাইন করা (Google Drive/Dropbox-এর মতো)

**Difficulty:** Advanced (Capstone)
**Estimated length:** 25-30 min
**Prerequisites:**
- [একটি Distributed File Storage System ডিজাইন করা (Google Drive/Dropbox-এর মতো)](../../Module-03-Databases-and-Storage/13-database-replication/README.md)
- [একটি Distributed File Storage System ডিজাইন করা (Google Drive/Dropbox-এর মতো)](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md)
- [একটি Distributed File Storage System ডিজাইন করা (Google Drive/Dropbox-এর মতো)](../../Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md)
- [একটি Distributed File Storage System ডিজাইন করা (Google Drive/Dropbox-এর মতো)](../../Module-04-Caching-and-Content-Delivery/18-cdn-explained/README.md)
- [একটি Distributed File Storage System ডিজাইন করা (Google Drive/Dropbox-এর মতো)](../../Module-05-Messaging-and-Asynchronous-Systems/22-event-driven-architecture/README.md)
- [একটি Distributed File Storage System ডিজাইন করা (Google Drive/Dropbox-এর মতো)](../../Module-06-Distributed-Systems-Concepts/27-consensus-algorithms-paxos-and-raft/README.md)
- [একটি Distributed File Storage System ডিজাইন করা (Google Drive/Dropbox-এর মতো)](../../Module-06-Distributed-Systems-Concepts/29-data-consistency-models-and-idempotency/README.md)

## Learning Objectives

- একটি cloud file-sync product-কে একটি **metadata service** এবং একটি **block/chunk storage service**-এ ভেঙে ফেলুন, এবং ব্যাখ্যা করুন কেন এই বিভাজনই মূল design সিদ্ধান্ত।
- Google Drive/Dropbox-এর scale-এ একটি service-এর জন্য storage, bandwidth, এবং request-rate সংখ্যা estimate করুন।
- Database sharding ও replication (Module 3), consensus algorithm (Module 6), event-driven architecture (Module 5), এবং CDN (Module 4)-কে file storage সমস্যার নির্দিষ্ট অংশে প্রয়োগ করুন।
- Chunking, content-addressable storage, এবং delta sync ব্যাখ্যা করুন — যেগুলো বড় ফাইলের sync-কে efficient করে তোলার মূল mechanism।
- Metadata বনাম blob data-র জন্য strong ও eventual consistency-র মধ্যে trade-off নিয়ে, এবং offline edit-এর জন্য conflict resolution নিয়ে যুক্তি দিন।

## Script

### Hook/Intro

"আপনি আপনার laptop-এর Drive folder-এ একটি 2 GB video ফেলে দিলেন। বিশ সেকেন্ড পর, সেটা আপনার phone-এ, tablet-এ, এবং পৃথিবীর অন্য প্রান্তে থাকা আপনার সহকর্মীর laptop-এ চলে এসেছে -- আর যদি আপনি একটি মাত্র frame পরিবর্তন করেন, তাহলে পুরো file আবার upload হয় না। এক বিলিয়ন user এবং exabyte-স্কেল data-র ক্ষেত্রে এটা কীভাবে কাজ করে? আজ আমরা একটি distributed file storage এবং sync system design করছি -- Google Drive, Dropbox, বা OneDrive-এর কথা ভাবুন। এটি একটি capstone সমস্যা: এটি আগের module-গুলো থেকে sharding, replication, consensus, event-driven messaging, এবং CDN-কে একটি system-এ একত্রিত করে। চলুন এটাকে একটা আসল interview-র মতো ধরে নিয়ে ধাপে ধাপে সমাধান করি।"

### ধাপ ১: Requirements স্পষ্ট করা

"কিছু design করার আগে, আমি interviewer-এর সাথে scope নির্ধারণ করে নেব।

**Functional requirements:**
- User-রা যেকোনো size-এর file **upload এবং download** করতে পারবে, 1 KB-এর একটি text file থেকে শুরু করে 50+ GB-এর একটি video archive পর্যন্ত।
- File **স্বয়ংক্রিয়ভাবে একাধিক device জুড়ে sync হয়** -- laptop-এ edit করুন, কয়েক সেকেন্ডের মধ্যে phone-এ update দেখুন।
- System **file version history** রাখে, যাতে user আগের কোনো version-এ ফিরে যেতে পারে।
- User-রা read/write/owner permission সহ অন্যদের সাথে **file এবং folder share** করতে পারবে।
- Client-গুলোর **offline edit** সমর্থন করা উচিত -- network connection ছাড়াই কাজ চালিয়ে যাওয়া যাবে, এবং reconnect করার পর পরিবর্তনগুলো reconcile হবে।

**Non-functional requirements:**
- **Durability**-ই সবচেয়ে বড় অগ্রাধিকার -- আমরা industry-standard **99.999999999% (eleven nines)** annual durability-র কথা বলছি। একজন user-এর file হারানো একটি অগ্রহণযোগ্য failure mode, system মাঝে মাঝে ধীর হলেও তা মেনে নেওয়া যায়।
- Upload, download, এবং metadata read-এর জন্য **high availability** -- লক্ষ্য প্রায় 99.9% availability।
- Chunking-এর মাধ্যমে **বড় file দক্ষভাবে সামলানো** -- এক byte পরিবর্তনের জন্য কখনোই পুরো file আবার transfer করা উচিত না।
- Delta sync-এর মাধ্যমে **bandwidth efficiency** -- শুধু পরিবর্তিত byte-গুলোই network-এ যাওয়া উচিত।
- আমরা স্পষ্টভাবে real-time collaborative co-editing-কে (যা একটি ভিন্ন সমস্যা, Google Docs-এর operational-transform/CRDT model-এর কাছাকাছি) কম অগ্রাধিকার দেব -- আমরা file-level sync-এর উপর focus করছি।"

### ধাপ ২: Capacity Estimation

"চলুন এতে বাস্তব সংখ্যা বসাই, যাতে আমাদের design সিদ্ধান্তগুলো ভিত্তিসম্পন্ন হয়।

- **Users:** মোট 500 million user, এর মধ্যে প্রায় 100 million daily active user (DAU)।
- **User প্রতি storage:** গড়ে প্রতি user 5 GB storage ব্যবহার করে (free + paid tier মিলিয়ে)। তার মানে `500M x 5GB = 2.5 exabytes` মোট blob storage। 11-nines durability-র জন্য আমরা সাধারণত data center জুড়ে triple-replicate বা erasure-code করি, তাই provisioned raw capacity আসলে তার 3-5x -- ধরুন 8-12 exabytes।
- **দৈনিক upload:** প্রতিটি DAU যদি গড়ে দিনে প্রায় 50 MB touch/upload করে (নতুন file এবং edit মিলিয়ে), তাহলে `100M x 50MB = 5 PB/day` নতুন/পরিবর্তিত data প্রতিদিন ingest হচ্ছে।
- **Chunking:** আমরা file-গুলোকে fixed-size **4 MB chunk**-এ ভাগ করি। একটি 2 GB file মানে প্রায় 500 chunk। এটাই storage, deduplication, এবং delta transfer-এর একক (unit)।
- **Metadata record:** প্রতিটি file/folder একটি metadata row, এবং প্রতিটি chunk reference-ও একটি row। যদি গড় user-এর 2,000টি file থাকে, তাহলে `500M x 2,000 = 1 trillion` file-metadata record, প্লাস কয়েক trillion chunk-reference record যা file-গুলোকে তাদের chunk hash-এর সাথে map করে। প্রতিটি metadata record ছোট -- হয়তো 200-500 byte -- তাই মোট metadata volume কয়েক দশ terabyte-এর মধ্যে, যা exabyte-scale blob data-র চেয়ে সম্পূর্ণ ভিন্ন একটি scaling সমস্যা।
- **Request rate:** metadata operation (folder listing, sync status check, permission resolve) ঘন ঘন হয় এবং latency-sensitive -- peak-এ globally 500K-1M requests/sec অনুমান করা যায়। Blob storage request (আসল chunk upload/download) প্রতি user হিসেবে অনেক কম ঘন ঘন হয় কিন্তু byte-এর দিক থেকে অনেক ভারী -- হয়তো 50K-100K requests/sec, কিন্তু প্রতিটি megabyte-এর পরিমাণে data move করে।

এই বিভাজন -- **metadata-র জন্য high QPS, ছোট payload** বনাম **blob-এর জন্য কম QPS, বিশাল payload** -- এটাই ঠিক কারণ কেন আমরা তাদের সম্পূর্ণ ভিন্ন scaling strategy সহ দুটি আলাদা service হিসেবে architect করি।"

### ধাপ ৩: High-Level Design

"High level-এ, flow-টা এরকম দেখায়:

1. **Sync client**: user-এর device-এ চলা একটি background agent যা একটি local folder watch করে, একটি local embedded database (file path, chunk hash, এবং sync state-এর index) maintain করে, এবং filesystem event বা periodic scan ব্যবহার করে পরিবর্তন শনাক্ত করে।
2. **API Gateway / Load Balancer**: সব client traffic-এর entry point, যা TLS termination, auth, এবং সঠিক backend service-এ request route করার কাজ করে।
3. **Metadata Service**: file/folder tree, version, permission, এবং একটি file version থেকে তার chunk hash-এর list-এর mapping-এর source of truth। এটি একটি sharded, replicated database দ্বারা backed।
4. **Chunk/Block Storage Service**: একটি object store (S3-এর মতো blob storage ভাবুন) যা তাদের content hash দ্বারা addressed immutable chunk store করে। এখানেই আসলে exabyte-গুলো থাকে।
5. **Notification Service**: একটি event-driven pub/sub layer যা user-এর *অন্যান্য* device-কে বলে, 'হেই, file X পরিবর্তন হয়েছে, গিয়ে sync করো,' যাতে update near real time-এ ছড়িয়ে পড়ে, client polling-এর উপর নির্ভর না করে।

Upload path: client locally file chunk করে, প্রতিটি chunk hash করে, metadata service-কে জিজ্ঞেস করে কোন chunk ইতিমধ্যে জানা আছে (dedup check), শুধু missing chunk-গুলো block storage-এ upload করে, তারপর নতুন file version (chunk hash-এর একটি ordered list) metadata service-এ commit করে। এরপর metadata service একটি 'file changed' event publish করে, এবং notification service সেটা user-এর অন্যান্য connected device-এ fan out করে, যারা delta pull করে।"

### ধাপ ৪: মূল Component-গুলোতে Deep Dive

"চলুন সেই অংশগুলোতে zoom in করি যা এই system-কে scale-এ আসলে কাজ করায়।

**Chunking এবং content-addressable storage।** প্রতিটি file fixed-size chunk-এ (~4 MB) ভাগ করা হয়, এবং প্রতিটি chunk হ্যাশ করা হয় (যেমন, SHA-256) একটি content address তৈরি করতে। আমরা block storage service-এ সেই hash দিয়ে key করা chunk store করি -- এটাই **content-addressable storage (CAS)**। এখান থেকে দুটো বিশাল সুবিধা পাওয়া যায়: (1) **deduplication** -- যদি দুইজন user একই identical PDF upload করে, বা একজন user একই ছবি দুইবার upload করে, তাহলে chunk-গুলো ইতিমধ্যে present আছে এবং আমরা পুরো upload skip করি, যা বলা 2.5 exabyte storage-এ বিশাল গুরুত্বপূর্ণ; এবং (2) **efficient sync** -- যখন user একটি বড় file edit করে, শুধু যেসব chunk আসলে পরিবর্তিত হয়েছে সেগুলোর নতুন hash তৈরি হয়, তাই আমরা শুধু delta upload এবং download করি, পুরো file নয়। এটাই `rsync`-style delta encoding-এর পেছনের একই principle।

**Metadata database: sharding এবং replication (Module 3)।** এক trillion-plus metadata row এবং ভারী read/write QPS নিয়ে, একটি single database এটা ধরে রাখতে পারবে না। আমরা **user ID** (বা shared drive-এর জন্য `file_owner_id`) দিয়ে shard করি, যাতে একজন নির্দিষ্ট user-এর সব file একই shard-এ পড়ে -- এটা 'list my files' এবং 'sync my account' query-গুলোকে cluster জুড়ে ছড়িয়ে না দিয়ে একটি shard-এ local রাখে। প্রতিটি shard durability এবং read scaling-এর জন্য একটি replicated database (যেমন, প্রতি shard-এ 3-5টি replica) -- সরাসরি Module 3-এর replication playbook থেকে।

**Metadata consistency-র জন্য Consensus (Module 6)।** Metadata হারানো বা corrupt হওয়া file নিজেই হারানোর মতোই খারাপ -- একটি dangling chunk reference একটি file-কে unrecoverable করে তোলে। তাই একটি metadata shard-এ write একটি **Raft (বা Paxos)** consensus group-এর মধ্য দিয়ে যায়: একটি write (যেমন একটি নতুন file version commit করা) শুধু তখনই acknowledge হয় যখন সেই shard-এর replica-গুলোর একটি majority durably সেটা apply করেছে। এটা আমাদের strong consistency এবং shard-এর মধ্যে automatic leader failover দেয়, যা critical কারণ metadata হলো blob store-এর মধ্যে 'source of truth' pointer।

**Cross-device sync-এর জন্য Event-driven architecture (Module 5)।** Metadata service একটি পরিবর্তন commit করার পর, এটি একটি message broker-এ একটি event (`file.updated`, `file.shared`, `version.created`) publish করে। Notification service এই event-গুলোতে subscribe করে এবং একটি persistent connection (WebSocket বা long-poll)-এর মাধ্যমে user-এর অন্যান্য online device-এ হালকা 'কিছু পরিবর্তন হয়েছে, check করো' ping push করে। এই pub/sub, event-driven approach-এর মানে হলো device-গুলোকে ক্রমাগত metadata service poll করতে হয় না -- এটা push-based, low-latency, এবং 'একটি পরিবর্তন শনাক্ত করা'-কে 'প্রতিটি device-কে জানানো' থেকে decouple করে।

**দ্রুত download-এর জন্য CDN (Module 4)।** publicly shared file বা কোনো organization-এর মধ্যে ব্যাপকভাবে shared file-এর জন্য, আমরা block storage service-এর সামনে একটি **CDN** বসাই। একটি shared video বা একটি company-wide onboarding PDF viewer-দের কাছাকাছি edge location-এ cache হয়ে যায়, যাতে একই popular chunk-এর বারবার download origin storage-এ hit না করে -- এটা 'hot' shared content-এর জন্য latency এবং origin load নাটকীয়ভাবে কমায়, যখন private, কম-access হওয়া file সরাসরি object store থেকেই serve হতে থাকে।"

### ধাপ ৫: Bottleneck এবং Trade-off

"কোনো design-ই তার trade-off না বললে সম্পূর্ণ নয়।

- **Concurrent/offline edit-এর জন্য Conflict resolution।** যদি একজন user offline অবস্থায় তাদের laptop-এ একটি file edit করে, এবং সেই একই file তাদের phone-এও edit করে, তাহলে আমরা দুটো divergent version পাই। আমরা নির্বিচারে binary file নীরবে merge করতে পারি না, তাই সাধারণ approach হলো **conflict copy সহ last-writer-wins** -- system উভয় version রাখে এবং user manually reconcile করার জন্য একটি 'conflicted copy' file তৈরি করে, যেমনটা আজ Dropbox করে থাকে। আমরা file version number বা vector clock (Module 6-এর territory) ব্যবহার করি একটি conflict ঘটেছে তা শনাক্ত করতে, data নীরবে overwrite না করে।
- **Strong বনাম eventual consistency (CAP theorem)।** Metadata-র জন্য, আমরা **strong consistency**-র দিকে ঝুঁকি (Raft-backed write-এর মাধ্যমে) কারণ একটি inconsistent file tree বিভ্রান্তিকর এবং data loss ঘটাতে পারে। Blob/chunk storage-এর জন্য, আমরা **eventual consistency এবং availability**-র দিকে ঝুঁকি -- যদি একটি নতুন uploaded chunk সব replica-তে propagate হতে এক সেকেন্ড বেশি লাগে, সেটা higher write throughput এবং availability-র জন্য একটি গ্রহণযোগ্য trade-off, যেহেতু chunk যেকোনোভাবেই immutable এবং content-addressed (এখানে কোনো 'stale value' সমস্যা নেই, শুধু 'এখনো সব জায়গায় replicate হয়নি' সমস্যা আছে)।
- **Storage cost optimization।** Deduplication (CAS-এর মাধ্যমে) আমাদের প্রথম lever। দ্বিতীয়টি হলো **storage tiering**: প্রায় 90 দিন access না হওয়া file-গুলো সস্তা, বেশি-latency-র cold storage-এ (যেমন S3 Glacier-class tier) move করা হয়, যখন hot/recent file দ্রুত storage-এই থাকে। Version history-ও একটি cost driver -- storage growth সীমাবদ্ধ রাখতে আমরা retained version-এর সংখ্যা বা তাদের retention window সীমিত করি।
- **Resumable বড় upload।** যেহেতু আমরা chunk-by-chunk upload করি, একটি অস্থির connection-এর উপর একটি 50 GB upload failure হলে শূন্য থেকে restart করার দরকার নেই -- client track করে কোন chunk acknowledge হয়েছে এবং শুধু missing chunk-গুলো retry করে। এই chunk-level checkpointing হলো ধাপ ৪-এ আমাদের নেওয়া chunking সিদ্ধান্তের একটি সরাসরি, ব্যবহারিক সুফল।"

### সারসংক্ষেপ

"সংক্ষেপে বলতে গেলে: আমরা সমস্যাটাকে একটি metadata service (sharded, replicated, strong consistency-র জন্য consensus-backed) এবং একটি block storage service (content-addressable, deduplicated, eventually consistent, hot content-এর জন্য CDN-fronted)-এ ভাগ করেছি। Content hash সহ chunking আমাদের একটি মাত্র mechanism থেকে deduplication, delta sync, এবং resumable upload -- সবকিছুই দেয়। Event-driven pub/sub polling ছাড়াই near real time-এ device-গুলোকে sync রাখে। এবং আমাদের trade-off-গুলো -- conflict copy সহ last-writer-wins, strong metadata / eventual blob consistency, এবং storage tiering -- Dropbox এবং Google Drive-এর মতো production system-গুলো আসলে যে সিদ্ধান্তগুলো নেয় তা প্রতিফলিত করে।"

### এরপর কী?

"এটা মূল distributed systems concept-গুলোকে বাস্তব product-এ প্রয়োগ করা আমাদের case-study series-এর সমাপ্তি টানছে। এখান থেকে, নিজে নিজে এই design-এর একটি variant sketch করার চেষ্টা করুন -- এর উপর real-time collaborative editing যোগ করা হলে আপনার উত্তর কীভাবে পরিবর্তিত হবে, যেমন Google Docs-এর মতো? পরবর্তীতে practice করার জন্য এটা একটি চমৎকার follow-up সমস্যা।"

## Key Takeaways

- **Metadata service** (ছোট record, high QPS, strongly consistent, user/file ID দ্বারা sharded, Raft-backed replication সহ)-কে **block storage service** (বিশাল payload, content-addressable, deduplicated, eventually consistent) থেকে আলাদা করুন।
- **Chunking** (~4 MB chunk, content-hashed) হলো একমাত্র mechanism যা deduplication, delta sync, এবং resumable upload সম্ভব করে তোলে।
- **Consensus algorithm (Raft/Paxos)** metadata-র সঠিকতা রক্ষা করে; একটি chunk reference হারানো file হারানোর মতোই খারাপ।
- **Event-driven pub/sub**-ই multi-device sync-কে poll-based-এর বদলে instant অনুভব করায়।
- Shared/public file download-এর জন্য **CDN** গুরুত্বপূর্ণ, private per-user data-র জন্য নয়।
- Trade-off-গুলো ইচ্ছাকৃতভাবে asymmetric: file tree-র সঠিকতা যেখানে গুরুত্বপূর্ণ সেখানে strong consistency, chunk-এর immutability যেখানে নিরাপদ করে তোলে সেখানে eventual consistency এবং availability।
