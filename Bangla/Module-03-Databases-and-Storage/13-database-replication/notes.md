# নোটস: Database Replication

## সংজ্ঞাসমূহ (Definitions)

- **Replication** — একাধিক database node জুড়ে ডেটা কপি করা এবং বজায় রাখার প্রক্রিয়া যাতে একাধিক মেশিন একই ডেটা ধারণ করে, availability, durability, এবং read scaling-এর জন্য।
- **Leader / Master / Primary** — একটি নির্দিষ্ট topology-তে যে node write গ্রহণ করে।
- **Follower / Slave / Replica** — একটি node যা leader-এর ডেটার একটি কপি ধারণ করে এবং সাধারণত read সার্ভ করে।
- **Replication lag** — একটি write primary-তে পড়া এবং সেই write একটি replica-তে দৃশ্যমান হওয়ার মধ্যকার বিলম্ব। সাধারণত asynchronous replication-এর কারণে ঘটে।
- **Failover** — একটি ব্যর্থ primary/master সনাক্ত করা এবং তার জায়গা নেওয়ার জন্য একটি replica/follower promote করার প্রক্রিয়া।
- **Read-your-writes problem** — একটি consistency বাগ যেখানে একজন ব্যবহারকারী তার নিজের সাম্প্রতিক write দেখতে পান না কারণ সেটি এমন একটি replica থেকে read করা হয়েছে যা তখনো ধরতে পারেনি (replication lag-এর একটি লক্ষণ)।
- **Write conflict** — multi-leader সিস্টেমে, যখন দুটি node একই ডেটায় একযোগে write গ্রহণ করে এবং একে অপরের কাছে replicate করার পর মানগুলো একমত হয় না।
- **Last-write-wins (LWW)** — একটি conflict resolution কৌশল যা সবচেয়ে সাম্প্রতিক timestamp-এর write রাখে এবং অন্যটি বাতিল করে।
- **Vector clock** — একটি data structure যা node জুড়ে write-এর কার্যকারণ (causal) ক্রম ট্র্যাক করে, প্রকৃত concurrency বনাম একটি write অন্যটির কার্যকারণে অনুসরণ করছে কিনা তা সনাক্ত করতে ব্যবহৃত হয়।

## Master-Slave বনাম Master-Master

| মাত্রা | Master-Slave (Leader-Follower) | Master-Master (Multi-Leader) |
|---|---|---|
| Write scaling | সীমিত — সব write একটি master দিয়ে যায় | উন্নত — একাধিক node-এ write গ্রহণ করা হয় |
| Read scaling | শক্তিশালী — read ট্রাফিক শোষণ করতে follower যোগ করুন | শক্তিশালী — একইভাবে leader/follower যোগ করুন |
| Conflict handling | দরকার নেই — একজন writer, কোনো conflict নেই | প্রয়োজন — LWW, vector clocks, বা app-level merge দরকার |
| Complexity | কম — বোঝা ও পরিচালনা করা সহজ | বেশি — bidirectional replication + conflict resolution |
| Failover | একটি follower-এর স্পষ্ট promotion; সংক্ষিপ্ত write downtime সম্ভব | write-এর জন্য কোনো single point of failure নেই; অন্য master-রা সার্ভ করতে থাকে |
| সাধারণ ব্যবহারের ক্ষেত্র | Read-heavy অ্যাপ (blog, content site, বেশিরভাগ CRUD অ্যাপ) | Multi-region active-active অ্যাপ, collaborative/global low-latency write |

## Synchronous বনাম Asynchronous Replication

| মাত্রা | Synchronous | Asynchronous |
|---|---|---|
| Write acknowledgment | client-কে ack করার আগে replica(-দের) confirmation-এর জন্য অপেক্ষা করে | client-কে সাথে সাথে ack করে, ব্যাকগ্রাউন্ডে replicate করে |
| Durability | শক্তিশালী — write confirm হওয়ার আগেই replica-র কাছে ডেটা থাকে | দুর্বল — এমন একটি window থাকে যেখানে শুধু primary-র কাছেই ডেটা থাকে |
| Write latency | বেশি — প্রতি write-এ network round trip-এর মূল্য দেয় | কম — কোনো round trip অপেক্ষা নেই |
| Failure risk | primary মারা গেলেও কম ডেটা হারানোর ঝুঁকি | primary মারা গেলে replicate না হওয়া write-এর জন্য সম্ভাব্য ডেটা ক্ষতি |
| সাধারণ ব্যবহার | নির্বাচিতভাবে, গুরুত্বপূর্ণ write / একটি replica-র জন্য | বেশিরভাগ production সিস্টেমে বেশিরভাগ replica-র জন্য ডিফল্ট |

## গুরুত্বপূর্ণ সংখ্যা / সাধারণ নিয়ম (Key Numbers / Rules of Thumb)

- স্বাভাবিক লোডে asynchronous replication lag প্রায়ই মিলিসেকেন্ড হয় কিন্তু ভারী write লোড বা network সমস্যায় তা সেকেন্ডে (বা বেশি) বাড়তে পারে।
- একটি সাধারণ hybrid প্যাটার্ন: ১টি synchronous replica (durability-র জন্য) + N-টি asynchronous replica (read scaling এবং ভৌগোলিক বিস্তারের জন্য)।
- Read-heavy অ্যাপগুলো প্রায়ই ১০০:১ বা তার বেশি read:write অনুপাতে চলে — এটি একটি শক্তিশালী সংকেত যে master-slave read replica-ই সঠিক প্রথম scaling lever।

## দ্রুত সারসংক্ষেপ বুলেট (Quick Summary Bullets)

- Replication = availability + read scaling-এর জন্য node জুড়ে ডেটার একাধিক কপি।
- Sync replication durability-র বিনিময়ে latency দেয়; async গতি এবং replication lag তৈরির বিনিময়ে সামান্য durability ঝুঁকি নেয়।
- Master-Slave: একজন writer, অনেক reader; সহজ; master হারালে failover দরকার; read-your-writes problem-এর জন্য সতর্ক থাকুন।
- Master-Master: অনেক writer; multi-region active-active সক্ষম করে; একটি conflict resolution কৌশল দরকার (LWW বা vector clocks)।
- Replication read ভালোভাবে scale করে (এবং, master-master-এ, আংশিকভাবে write-ও); এটি মোট ডেটা আকার বা চূড়ান্ত write-throughput সীমা সমাধান করে না — সেটাই sharding-এর কাজ।
