# কেন এই বিষয়টি গুরুত্বপূর্ণ: Database Sharding & Partitioning

> **এক বাক্যে:** Replica দিয়ে আপনি সীমাহীন read scale করতে পারেন, কিন্তু প্রতিটি write তখনও একটি node-এই গিয়ে পড়ে — sharding-ই সেই দেয়াল পার হওয়ার একমাত্র উপায়, এবং এটাই একটি ডেটাবেসের সাথে করা সবচেয়ে ব্যয়বহুল, সবচেয়ে কম reversible কাজ।

## এই ধারণার আগে যে পৃথিবী ছিল

আপনি সবকিছু index করেছেন, read replica যোগ করেছেন, আক্রমণাত্মকভাবে cache করেছেন। Read ঠিকঠাক চলছে। কিন্তু আপনি সেকেন্ডে ৮০,০০০ row লিখছেন এবং primary saturated হয়ে গেছে। এর চেয়ে বড় মেশিন নেই, এবং write ঠিক করতে আপনি replica যোগ করতে পারবেন না — replica-গুলো একই write *পায়*, তাই তারা load একটুও কমায় না।

এদিকে আপনার table ৮ TB পার হয়ে গেছে। Backup নিতে একদিন লাগে। Index rebuild করতে একটা সপ্তাহান্ত লাগে। একটি schema migration একটি বহু-সপ্তাহের প্রজেক্ট। শুধু traffic নয়, dataset নিজেই একটি single machine-এর চেয়ে বড় হয়ে গেছে।

## এটি যেসব সমস্যার সমাধান করে

### ১. এমন একটি write সীমা যা replica বাড়াতে পারে না
**আপনি যা দেখেন:** Primary-র disk এবং CPU pinned হয়ে গেছে। Replica যোগ করলেও কিছু হচ্ছে না।

**কেন এটা ঘটে:** প্রতিটি write অবশ্যই primary-তে apply করতে হবে, এবং replica-গুলো একই write stream apply করে। Replication read scale করে, write নয় — এটাই এ সম্পর্কে বোঝার সবচেয়ে গুরুত্বপূর্ণ বিষয়।

**Sharding কীভাবে এটি সমাধান করে:** N-টি স্বাধীন primary জুড়ে key দিয়ে ডেটা ভাগ করুন। প্রতিটি shard মোটামুটি ১/N অংশ write পায় এবং তার নিজস্ব disk, CPU, এবং memory-র মালিক। Write throughput এখন shard-সংখ্যার সাথে বাড়ে।

### ২. একটি dataset যা কাজ করার জন্য অতিরিক্ত বড়
**আপনি যা দেখেন:** Backup, restore, migration, এবং index build — সবকিছুতেই এত সময় লাগে যে এগুলো নিরাপদে করা কার্যত অসম্ভব হয়ে যায়।

**কেন এটা ঘটে:** একটি single node-এ operational সময় data volume-এর সাথে সাথে বাড়ে।

**Sharding কীভাবে এটি সমাধান করে:** প্রতিটি shard একটি manageable আকারের। আপনি প্রতিটি shard আলাদাভাবে, সমান্তরালভাবে backup, migrate, এবং rebuild করেন, এবং একটি shard-এর সমস্যা সবার বদলে ১/N ব্যবহারকারীকে প্রভাবিত করে। Data-র আকারের সাথে সাথে blast radius-ও ছোট হয়ে যায়।

### ৩. Hot ডেটা এবং cold ডেটার মধ্যে প্রতিযোগিতা
**আপনি যা দেখেন:** এই মাসের ডেটার query ধীর, কারণ একই table এবং একই buffer pool-এ সেগুলো পাঁচ বছরের archived row-এর সাথে প্রতিযোগিতা করছে।

**কেন এটা ঘটে:** কতবার touch করা হয় তা নির্বিশেষে সব ডেটা সমানভাবে বিদ্যমান থাকে।

**Partitioning কীভাবে এটি সমাধান করে:** সময় অনুযায়ী range-partitioning সাম্প্রতিক ডেটাকে ছোট, hot partition-এ রাখে যা memory-তে থেকে যায়, যখন পুরনো partition ঠান্ডা হয়ে বসে থাকে। গত বছরের ডেটা ফেলে দেওয়া এখন একটি তাৎক্ষণিক partition drop হয়ে যায়, ছয় ঘণ্টা ধরে চলা এবং table-কে ফুলিয়ে তোলা একটি `DELETE`-এর বদলে।

### ৪. এমন একটি shard key বেছে নেওয়া যা সবকিছু নষ্ট করে দেয়
**আপনি যা দেখেন:** আপনি shard করেছেন, এবং একটি shard ৯৫% CPU-তে চলছে যখন অন্য নয়টি অলস বসে আছে। অথবা প্রতিটি query-কেই এখন সব দশটি shard-কে জিজ্ঞেস করে সবচেয়ে ধীরটির জন্য অপেক্ষা করতে হয়।

**কেন এটা ঘটে:** shard key সবকিছু নির্ধারণ করে। `country` দিয়ে shard করলে একটি দেশ প্রাধান্য বিস্তার করে। timestamp দিয়ে shard করলে *সব* বর্তমান write সবচেয়ে নতুন shard-এ গিয়ে পড়ে — গঠনগতভাবেই একটি hotspot। এমন একটি key দিয়ে shard করলে যা দিয়ে আপনার query filter করে না, প্রতিটি query একটি scatter-gather হয়ে যায়।

**এই বিষয়টি কীভাবে এটি সমাধান করে:** এটি shard key-কে কেন্দ্রীয় সিদ্ধান্ত বানায়, তিনটি মানদণ্ডের বিপরীতে বেছে নেওয়া: সমান বণ্টন, আপনার প্রধান query pattern-এর সাথে সামঞ্জস্য, এবং সময়ের সাথে স্থিতিশীলতা। এটা ঠিক করা কাজের বেশিরভাগ অংশ; এটা ভুল করার মানে হলো resharding, যেটাই সবচেয়ে কষ্টের অংশ।

## যে মূল্য আপনাকে দিতে হয়

Scale ছাড়া বাকি সবকিছুতে sharding একটি প্রকৃত architectural অবনতি, এবং সেটাই সৎ দৃষ্টিভঙ্গি:

- **Cross-shard query ধীর বা অসম্ভব।** shard key অন্তর্ভুক্ত না করা একটি query-কে অবশ্যই প্রতিটি shard-এ যেতে হবে এবং ফলাফল merge করতে হবে — সবচেয়ে ধীর shard-এর মতোই ধীর, এবং paginate বা sort করা অনেক কঠিন।
- **Cross-shard join কার্যত থাকেই না।** আপনি denormalize করেন, ডেটা duplicate করেন, বা application-এ join করেন।
- **Cross-shard transaction কার্যত থাকেই না।** ACID প্রতি-shard ভিত্তিতে। shard জুড়ে বিস্তৃত যেকোনো কিছুর জন্য two-phase commit (ধীর, fragile) বা saga (eventually consistent, জটিল) দরকার।
- **Resharding নিষ্ঠুর।** shard key পরিবর্তন করা বা shard যোগ করার মানে হলো live অবস্থায় বিশাল পরিমাণ ডেটা সরানো। Consistent hashing কষ্ট কমায় কিন্তু পুরোপুরি সরায় না।
- **Operational বহুগুণন।** দশটি shard, প্রতিটির নিজস্ব replica সহ, মানে monitor, backup, patch, এবং fail over করার জন্য কয়েক ডজন node। প্রতিটি operation এখন একটি fleet operation।
- **Hotspot থেকেই যায়।** একটি ভালো key থাকলেও, একজন celebrity ব্যবহারকারী বা একটি viral আইটেম একটি single shard-কে অভিভূত করে দিতে পারে।

**এই সবকিছুর কারণেই, sharding সবার শেষে করা উচিত।** আগে ঠিকমতো index করুন, cache করুন, replica যোগ করুন, পুরনো ডেটা archive করুন, hardware upgrade করুন, এবং সার্ভিস অনুযায়ী ভাগ করুন। দলগুলো নিয়মিতভাবে প্রয়োজনের বছরখানেক আগেই shard করে ফেলে এবং এমন একটি scale-এর জন্য প্রতিদিন complexity-র মূল্য দেয় যা কখনও আসেই না।

## কখন এটি দরকার — এবং কখন নয়

| Shard করুন যখন | অন্য কিছু করুন যখন |
|---|---|
| Write throughput একটি node-এর সীমা ছাড়িয়ে গেছে এবং আপনি tuning শেষ করে ফেলেছেন | Read bottleneck হচ্ছে → replica এবং caching |
| Dataset backup বা migrate করার জন্য অতিরিক্ত বড় | Query ধীর হচ্ছে → index |
| আপনার একটি স্বাভাবিক, উচ্চ-cardinality partition key আছে (user ID, tenant ID) | Growth মূলত পুরনো ডেটার → archive বা time-partition |
| আপনার প্রতি-tenant বা প্রতি-region ডেটা isolation দরকার | আপনি এখনও সাধারণ hardware-এ আছেন — আগে scale up করুন |

## Interview-এ কেন এটি আসে

Internet scale-এর যেকোনো design এমন একটি বিন্দুতে পৌঁছায় যেখানে একটি single ডেটাবেস ডেটা ধরে রাখতে পারে না — news feed, chat, ride-sharing, file storage সবই এমন হয়। Interviewer-রা শুনতে চান: shard key কী, *কেন সেই key*, এর কারণে কোন query ব্যয়বহুল হয়ে যায়, আপনি hotspot কীভাবে সামলান, এবং shard যোগ করলে কী হয়। একটি shard key নাম দিয়ে সাথে সাথেই এটা যেসব query-কে অস্বস্তিকর করে তোলে তা চিহ্নিত করা একটি শক্তিশালী সংকেত। কোনো পরিণতির আলোচনা ছাড়াই "আমরা user ID দিয়ে shard করব" বলা তা নয়।

## এটি কীভাবে সম্পর্কিত

Sharding হলো **replication**-এর read scaling-এর write-scaling সমতুল্য, এবং production সিস্টেমগুলো এদুটোকে একত্রিত করে (প্রতিটি shard নিজেই replicated)। **Consistent hashing** হলো key-গুলোকে shard-এর সাথে map করার standard টেকনিক যাতে একটি node যোগ করলে ন্যূনতম ডেটা সরে। Cross-shard transaction হারানোই ঠিক যে কারণে **distributed transaction (2PC এবং Saga)** বিদ্যমান। **CAP theorem** প্রযোজ্য কারণ একটি sharded cluster হলো partition সহ একটি distributed system। এবং join আর সম্ভব না হওয়ার পর **denormalization** প্রায় অনিবার্য হয়ে ওঠে।

**পরবর্তী:** [CAP Theorem & PACELC](../15-cap-theorem-and-pacelc/why.md) — যে তত্ত্ব আপনি এইমাত্র যেসব trade-off-এর মুখোমুখি হলেন তা ব্যাখ্যা করে।
