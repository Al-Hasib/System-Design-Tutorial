# এই বিষয়টি কেন গুরুত্বপূর্ণ: Distributed Locking (Redlock, ZooKeeper & etcd)

> **এক বাক্যে:** একটি mutex কাজ করে কারণ সব thread memory এবং একটি clock শেয়ার করে; একাধিক machine জুড়ে এর কোনোটিই থাকে না, আর এই কারণেই distributed lock সঠিকভাবে তৈরি করার চেয়ে সূক্ষ্মভাবে, বিপজ্জনকভাবে ভুল করা অনেক সহজ।

## এই ধারণার আগের জগৎ

আপনি একটি nightly billing job চালান। এটি আপনার application-এর ভেতরে একটি schedule অনুযায়ী চলে। আপনি বারোটি instance-এ scale করেন। এখন billing job বারোবার চলে, এবং প্রতিটি customer-কে বারোবার charge করা হয়।

স্পষ্ট সমাধান: "প্রথমে একটি lock নাও।" তাই প্রতিটি instance Redis-এ `SET lock:billing my-id NX EX 60` করে, এবং শুধু বিজয়ীই চালায়।

এটি সঠিক মনে হয়, আর এভাবেই এটি ব্যর্থ হয়। Instance A lock নেয় এবং কাজ শুরু করে। একটি দীর্ঘ garbage-collection pause এটিকে ৯০ সেকেন্ডের জন্য জমিয়ে দেয়। Lock-টি ৬০ সেকেন্ডে expire হয়ে যায়। Instance B এটি acquire করে এবং billing শুরু করে। Instance A জেগে ওঠে — কোনো ধারণা ছাড়াই যে কতটা সময় পেরিয়ে গেছে — এবং billing চালিয়ে যায়, বিশ্বাস করে যে সে এখনও lock ধরে আছে। দুটি instance, উভয়েই নিশ্চিত যে তারাই exclusive owner, উভয়েই লিখছে।

সেই sequence-এ আপনার code-এ কোনো bug নেই। এটি এমন একটি system-এ timeout-সহ একটি lock-এর default আচরণ যেখানে process pause হতে পারে এবং clock drift করতে পারে।

## এটি যেসব সমস্যার সমাধান করে

### ১. এমন কাজের duplicate execution যা ঠিক একবার চলা উচিত
**আপনি যা দেখেন:** N-instance fleet-এ scheduled job N বার চলছে। Duplicate charge, duplicate email, duplicate report generation।

**এটি কেন ঘটে:** Horizontal scaling সবকিছু replicate করে, এমনকি এমন জিনিসও যা implicitly singleton ছিল।

**Distributed locking এটি কীভাবে সমাধান করে:** ঠিক একটি instance lock acquire করে এবং কাজটি করে; বাকিরা এটি skip করে। একই mechanism singleton দায়িত্বের জন্য leader election প্রদান করে — একটি scheduler, একটি compaction coordinator, একটি cluster manager।

### ২. একটি shared বাহ্যিক resource-এর concurrent modification
**আপনি যা দেখেন:** দুইজন worker একই queue item process করে, অথবা দুটি service object storage-এ একই file লেখে, এবং ফলাফল হয় interleaved গোলমাল।

**এটি কেন ঘটে:** resource-টির নিজের কোনো transactional guarantee নেই। একটি database একটি row রক্ষা করতে পারে; S3-এর একটি file বা একটি third-party API পারে না।

**Locking এটি কীভাবে সমাধান করে:** এটি এমন কিছুর access-কে serialize করে যার নিজস্ব কোনো built-in serialization নেই।

### ৩. Lock যা holder এখনও কাজ করার সময় expire হয়ে যায়
**আপনি যা দেখেন:** ওপরে বর্ণিত failure — একই সাথে দুইজন holder, নীরবে।

**এটি কেন ঘটে:** একটি lock-এর অবশ্যই একটি TTL থাকতে হবে, নাহলে একজন crash হওয়া holder system-কে চিরকাল আটকে রাখবে। কিন্তু একটি TTL মানে হলো lock-টি expire হয়ে যেতে পারে যখন একজন জীবিত-কিন্তু-pause-করা holder তখনও বিশ্বাস করে যে সে এটি ধরে আছে। এর কোনো উপায় নেই: **আপনি একই সাথে crash-safety এবং এই guarantee পেতে পারেন না যে holder জানে সে এখনও lock ধরে আছে।**

**Fencing token এটি কীভাবে সমাধান করে:** lock service প্রতিটি grant-এর সাথে একটি monotonically increasing token ফেরত দেয়। Holder প্রতিটি write-এ সেই token পাস করে, এবং storage system এমন যেকোনো write প্রত্যাখ্যান করে যা এটি দেখা সর্বোচ্চ token-এর চেয়ে নিচু একটি token বহন করে। Instance A token 33 নিয়ে জেগে ওঠে, instance B ইতিমধ্যে token 34 দিয়ে লিখছে, এবং A-এর write প্রত্যাখ্যাত হয়। এটাই একমাত্র নির্মাণ যা প্রকৃতপক্ষে নিরাপদ, এবং এর জন্য প্রয়োজন যে *downstream system* নিজেও অংশগ্রহণ করুক — এই কারণেই অনেক বাস্তব deployment সত্যিকারের mutual exclusion অর্জন করতে পারেই না এবং এর বদলে idempotency-র ওপর নির্ভর করতে হয়।

### ৪. Lock যা failover-এ হারিয়ে যায়
**আপনি যা দেখেন:** একটি Redis primary এবং replica-সহ, primary একটি lock grant করে এবং replicate করার আগেই crash করে। Replica-টি lock-এর কোনো record ছাড়াই promote হয়, এবং এটি অন্য কাউকে grant করে দেয়।

**এটি কেন ঘটে:** Asynchronous replication মানে হলো lock-এর অস্তিত্ব grant করার মুহূর্তে durable ছিল না।

**Consensus-ভিত্তিক lock service এটি কীভাবে সমাধান করে:** ZooKeeper এবং etcd একটি consensus protocol-এর মাধ্যমে lock state commit করে, তাই acknowledge করার আগে এটি একটি majority দ্বারা সম্মত হয়। একটি failover এটিকে হারাতে পারে না। এরা ephemeral node এবং session-এর সাথে যুক্ত lease-ও প্রদান করে — যখন holder-এর session মারা যায়, lock-টি স্বয়ংক্রিয়ভাবে release হয়ে যায়, একটি নির্দিষ্ট timeout অনুমানের ওপর নির্ভর না করে। এটি একটি single Redis key-এর চেয়ে সত্যিকার অর্থেই শক্তিশালী, এবং এই কারণেই critical coordination একটি cache-এর বদলে etcd-তে থাকে।

## আপনি যে মূল্য দিচ্ছেন

- **Lock serialize করে, আর serialization হলো scaling-এর বিপরীত।** প্রতিটি lock নির্মাণগতভাবেই একটি bottleneck। একটি hot path-এর চারপাশে একটি coarse lock আপনার পুরো fleet-এর সুবিধা মুছে দিতে পারে।
- **Lock service একটি hard dependency।** যদি এটি ডাউন থাকে, কাজ বন্ধ হয়ে যায়। Fail open করা মানে duplicate execution; fail closed করা মানে একটি outage। আপনাকে সচেতনভাবে বেছে নিতে হবে।
- **Redlock বিতর্কিত।** Multi-node Redis locking algorithm-টি একটি সুপরিচিত technical বিতর্কের বিষয় হয়ে উঠেছে যে এটি clock drift এবং process pause-এর অধীনে প্রকৃত safety প্রদান করে কিনা। ব্যবহারিক শিক্ষাটি এই নয় যে কোন পক্ষ সঠিক: এটি হলো, **যদি correctness সত্যিকার অর্থেই গুরুত্বপূর্ণ হয়, একটি consensus-backed service এবং fencing token ব্যবহার করুন; যদি আপনি শুধু duplicate কাজ optimize করে দূর করতে চান, একটি সাধারণ Redis lock-ই ঠিক আছে।** আপনি কোন পরিস্থিতিতে আছেন তা নিয়ে স্পষ্ট থাকাই আসল দক্ষতা।
- **প্রতিটি acquisition-এ latency।** প্রতি lock-এ একটি consensus round trip সস্তা নয়, যা সংকীর্ণভাবে এবং সংক্ষিপ্তভাবে lock করার আরেকটি কারণ।
- **Deadlock এবং lock leak।** একজন holder যে release না করে crash করে, অথবা দুইজন holder যারা ভিন্ন ক্রমে lock নেয়, একটি single process-এর মতোই একই সমস্যা তৈরি করে — শুধু detection ধীর গতিতে হয়।

**এবং সবচেয়ে শক্তিশালী পদক্ষেপ প্রায়ই lock-টিকে সম্পূর্ণভাবে এড়িয়ে যাওয়া।** একটি idempotent operation-এর কোনো mutual exclusion-এর প্রয়োজন নেই। Database-এ একটি conditional atomic update (`UPDATE ... WHERE status = 'pending'`) database ইতিমধ্যে যে guarantee প্রদান করে তা ব্যবহার করে exclusion অর্জন করে। Key অনুযায়ী কাজ partition করা যাতে শুধুমাত্র একজন worker কখনো একটি নির্দিষ্ট key-এর মালিক হতে পারে, নির্মাণগতভাবেই contention দূর করে দেয়। এগুলোর পরেই একটি distributed lock-এর কাছে পৌঁছান, আগে নয়।

## কখন এটি প্রয়োজন — এবং কখন নয়

| আপনার একটি distributed lock প্রয়োজন যখন | অন্য কিছু ব্যবহার করুন যখন |
|---|---|
| ঠিক একটি instance-ই একটি action সম্পন্ন করতে পারে | operation-টি idempotent করা যেতে পারে |
| আপনার একটি singleton role-এর জন্য leader election প্রয়োজন | database একটি conditional update দিয়ে এটি প্রয়োগ করতে পারে |
| একটি shared বাহ্যিক resource-এর কোনো transaction সমর্থন নেই | কাজ এমনভাবে partition করা যায় যাতে owner-রা কখনো overlap না করে |
| Correctness প্রকৃত mutual exclusion-এর ওপর নির্ভর করে | আপনি শুধু duplicate কাজ কমাতে চান, প্রতিরোধ করতে নয় |

## এটি কেন Interview-এ দেখা যায়

Scheduled job, singleton worker, বা একটি shared resource-এর ওপর contention-সহ যেকোনো design এটি প্রকাশ করতে পারে। Interviewer সাধারণত পরীক্ষা করছেন আপনি কঠিন অংশটি জানেন কিনা — expiry সমস্যা। "আমি একটি Redis lock নেব" বলা হলো entry-level উত্তর। "আমি একটি TTL-সহ একটি lock নেব, কিন্তু শুধু একটি lock যথেষ্ট নয়, কারণ একটি GC pause TTL-কে expire হতে দিতে পারে যখন holder এখনও মনে করে সে এটির মালিক — তাই হয় আমি etcd-এর মতো একটি consensus-backed service-এর সাথে fencing token ব্যবহার করব, অথবা, বরং পছন্দনীয়ভাবে, আমি operation-টিকে idempotent বানাব যাতে একটি duplicate ক্ষতিকর না হয়" বলা একটি সত্যিকারের senior উত্তর।

## এটি কীভাবে সংযুক্ত

সঠিক distributed locking **consensus** (topic 27)-এর ওপর নির্মিত, যা etcd এবং ZooKeeper-কে এর জন্য বিশ্বস্ত করে তোলে। সহজ lock **Redis** (topic 19)-এর ওপর নির্মিত। **Idempotency** (topic 29) locking-এর বিকল্প এবং locking ব্যর্থ হলে safety net — উভয়ই। এটি **topic 37**-এর concurrency control-এর distributed সমতুল্য, **horizontal scaling** (topic 4)-এর কারণে প্রয়োজনীয় হয়ে ওঠে, এবং controller-দের মধ্যে leader election-এর জন্য **Kubernetes** (topic 44)-এর ভেতরে ক্রমাগত ব্যবহৃত হয়।

**পরবর্তী:** [Logical Clocks & Time in Distributed Systems](../41-logical-clocks-and-time-in-distributed-systems/why.md) — কেন আপনি এইমাত্র যে clock-এর ওপর ভরসা করলেন তা বিশ্বাস করা যায় না।
