# এই বিষয়টি কেন গুরুত্বপূর্ণ: LSM Trees vs B-Trees — Storage Engine Internals

> **এক বাক্যে:** "Cassandra write-এ দ্রুত আর Postgres read-এ দ্রুত" — এই দাবিটা বেশিরভাগ engineer পুনরাবৃত্তি করেন কিন্তু ব্যাখ্যা করতে পারেন না — ব্যাখ্যাটি হলো storage engine, এবং এটি জানলে database selection folklore থেকে যুক্তিতে রূপান্তরিত হয়।

## এই ধারণার আগে জগৎ কেমন ছিল

আপনি একই hardware-এ একই data দিয়ে দুটি database benchmark করলেন। একটি প্রতি সেকেন্ডে 200,000 write ingest করে; অন্যটি 20,000 সামলাতে পারে। দ্বিতীয়টি range query-র উত্তর তাৎক্ষণিকভাবে দেয়; প্রথমটি কখনো কখনো তার average-এর চেয়ে অনেক বেশি সময় নেয়। কোনোটিই ভালোভাবে engineer করা নয় — খারাপভাবেও না। তারা data যেখানে disk-এর সাথে মিলিত হয়, সেই স্তরে বিপরীত সিদ্ধান্ত নিয়েছে।

সেই স্তর বুঝা ছাড়া, database নির্বাচন নির্ভর করে এমন benchmark-এর উপর যা আপনি নিজে design করেননি, এবং এমন blog post-এর উপর যা অন্য কারো workload নিয়ে লেখা। আপনি *আপনার* access pattern-এ একটি system কেমন আচরণ করবে তা আগে থেকে বলতে পারবেন না, এবং তিন মাস পরে production-এ দেখা দেওয়া latency spike-টিও ব্যাখ্যা করতে পারবেন না।

## এটি যেসব সমস্যার সমাধান করে

### ১. Random write, যেখানে disk খারাপ পারফর্ম করে
**আপনি যা দেখেন:** একটি write-heavy workload — event, metric, log, sensor data — hardware-এর নির্ধারিত throughput-এর অনেক নিচেই disk I/O saturate করে ফেলছে।

**যে কারণে এটা ঘটে:** একটি B-Tree data-কে *in place* update করে। একটি row লেখা মানে তার page খুঁজে বের করা, পড়া, modify করা, এবং আবার লিখে ফেলা — প্রতিটি write-এর জন্য একটি random I/O, সাথে মাঝেমধ্যে page split। Random I/O-তেই storage device সবচেয়ে দুর্বল, বিশেষত spinning disk-এ, এবং SSD-তেও এটি sequential-এর তুলনায় উল্লেখযোগ্যভাবে খারাপ।

**LSM tree এটা কীভাবে সমাধান করে:** Write প্রথমে একটি in-memory structure এবং একটি append-only write-ahead log-এ যায়, তারপর সম্পূর্ণ sequentially immutable sorted file হিসেবে disk-এ flush হয়। Sequential write অনেক বেশি দ্রুত, এবং কোনো read-modify-write চক্র নেই। এই কারণেই Cassandra, RocksDB, LevelDB, HBase, এবং ScyllaDB-র অস্তিত্ব, এবং এই কারণেই এরা B-Tree engine-এর চেয়ে অনেক বেশি ingest করতে পারে।

### ২. Read, যেখানে অনেক জায়গা check করতে হয়
**আপনি যা দেখেন:** একটি LSM store-এ, এমন একটি key-এর জন্য lookup যা আসলে নেই, তা অপ্রত্যাশিতভাবে ব্যয়বহুল, এবং read latency আপনি যতটা চান তার চেয়ে বেশি ওঠানামা করে।

**যে কারণে এটা ঘটে:** একটি key-র data memtable-এ অথবা on-disk অনেকগুলো level-এর যেকোনো একটিতে থাকতে পারে। একটি read-কে সবগুলোই check করতে হতে পারে — এটাই কখনো in place update না করার মূল্য।

**engine এটা কীভাবে সমাধান করে:** প্রতিটি file-এর সামনে থাকা Bloom filter constant time-এ "নিশ্চিতভাবে এখানে নেই" উত্তর দেয়, যা বেশিরভাগ অপ্রয়োজনীয় disk read বাদ দিয়ে দেয়। Sparse index এবং block cache বাকিটা সামলায়। এটি বুঝলে বোঝা যায় কেন Bloom filter কোনো exotic optimization নয়, বরং প্রতিটি LSM engine-এর একটি load-bearing অংশ — এবং কেন LSM read amplification সামলানো যায়, কিন্তু কখনোই শূন্য হয় না।

### ৩. Production-এ অপ্রত্যাশিত latency spike
**আপনি যা দেখেন:** p99 latency যা p50-এর দশ বা একশো গুণ, কোনো traffic correlation ছাড়াই বিস্ফোরণাত্মকভাবে দেখা দেয়।

**যে কারণে এটা ঘটে:** **Compaction।** LSM engine-কে পর্যায়ক্রমে sorted file merge করতে হয়, deleted ও overwritten row থেকে space পুনরুদ্ধার করতে এবং একটি read-কে কতগুলো file check করতে হয় তা সীমিত রাখতে। Compaction হলো I/O-heavy background কাজ, যা live traffic-এর সাথে প্রতিযোগিতা করে। এটি LSM store সম্পর্কে সবচেয়ে গুরুত্বপূর্ণ operational তথ্য, এবং এই কারণেই এরা একটি ছোট benchmark-এ চমৎকার দেখাতে পারে কিন্তু সপ্তাহের পর সপ্তাহ production write-এর পর ভিন্ন আচরণ করতে পারে।

**এই বিষয়টি এটা কীভাবে সমাধান করে:** এটি আপনাকে জানায় যে এই spike অন্তর্নিহিত, কোনো bug নয়, এবং নিয়ন্ত্রণগুলোর দিকে ইঙ্গিত করে — compaction strategy (size-tiered write throughput-কে favor করে, leveled read performance এবং space efficiency-কে favor করে), throughput throttling, এবং এর জন্য headroom প্রভিশনিং।

### ৪. Delete-এর পরেও ফিরে না আসা space
**আপনি যা দেখেন:** আপনি অর্ধেক row delete করলেন, কিন্তু disk usage কমল না। কিছু engine-এ এটি সাময়িকভাবে বাড়েও।

**যে কারণে এটা ঘটে:** LSM delete হলো **tombstone** — নতুন data হিসেবে লেখা marker। compaction উভয়কে সরিয়ে না ফেলা পর্যন্ত original row থেকে যায়। B-Tree-তে, এর প্রতিচ্ছবি সমস্যাটা ভিন্ন কিন্তু বাস্তব: deleted space file-এর ভেতরে free page হয়ে যায়, যা OS পুনরুদ্ধার করে না, এবং এর ফলে bloat হয়, যার জন্য `VACUUM` বা rebuild দরকার হয়।

**এই বিষয়টি এটা কীভাবে সমাধান করে:** এটি storage আচরণকে predictable করে তোলে। tombstone সম্পর্কে জানা একটি কুখ্যাত Cassandra failure mode-ও ব্যাখ্যা করে: অনেক delete জমিয়ে রেখে তারপর সেই অংশে range-scan করলে engine-কে বিপুল সংখ্যক tombstone পড়তে বাধ্য করে, এবং query time out হয়ে যায়।

## যে মূল্য আপনাকে দিতে হয়

প্রতিটি engine ভিন্ন এক ধরনের কর দেয়, এবং সৎ ব্যাখ্যা হলো আপনি আসলে বেছে নিচ্ছেন *কোন* amplification-টা আপনি দেবেন:

- **B-Tree write amplification-এর মূল্য দেয়।** প্রতিটি write একটি random in-place page update, সাথে একটি write-ahead log entry, তাই একটি logical write আসলে একাধিক physical write হয়ে যায়। বিনিময়ে: predictable low-variance read, sorted page-এর উপর efficient range scan, এবং strong transactional guarantee-র জন্য straightforward locking।
- **LSM tree read amplification এবং space amplification-এর মূল্য দেয়।** Read-কে একাধিক level check করতে হতে পারে; deleted এবং overwritten data compaction না হওয়া পর্যন্ত space দখল করে থাকে। বিনিময়ে: অত্যন্ত উচ্চ sequential write throughput এবং ভালো compression, কারণ immutable sorted block ভালোভাবে compress হয়।
- **Compaction tuning একটি প্রকৃত operational specialty।** আপনার workload-এর জন্য একটি strategy বেছে নেওয়া এবং tune করা চলমান কাজ, একবারের সেটিং নয়।
- **কোনোটিই সাধারণ উত্তর নয়।** একটি workload যা read-heavy, প্রচুর range scan এবং in-place update সহ, তা একটি B-Tree workload। একটি workload যা মূলত append-প্রধান এবং key lookup-নির্ভর, তা একটি LSM workload। বেশিরভাগ system-এই দুটোই থাকে, ভিন্ন ভিন্ন table-এ — যা company-প্রতি নয়, বরং dataset-প্রতি সিদ্ধান্ত নেওয়ার পক্ষে যুক্তি।

## কখন এটি প্রয়োজন — এবং কখন নয়

| LSM (Cassandra, RocksDB, ScyllaDB, HBase) যখন | B-Tree (Postgres, MySQL InnoDB, SQL Server) যখন |
|---|---|
| Write volume অত্যন্ত বেশি এবং append-এর মতো | Read প্রাধান্য পায়, বিশেষত range scan |
| Time series, event, log, metric, sensor data | Workload-টি transactional OLTP |
| Storage efficiency এবং compression গুরুত্বপূর্ণ | Latency predictable এবং low-variance হতে হবে |
| আপনি compaction থেকে p99 variance সামলাতে পারেন | আপনার strong multi-row transactional guarantee দরকার |

## Interview-এ এটি কেন গুরুত্বপূর্ণ হয়ে ওঠে

এটি senior level-এ একটি শক্তিশালী differentiator। "কেন এখানে Cassandra?" জিজ্ঞেস করা হলে, folklore উত্তর হলো "এটি scale করে।" Engineering উত্তর হলো: "workload-টি append-heavy time series, key-based read সহ, এবং একটি LSM engine সেই write-কে sequential I/O-তে রূপান্তরিত করে, যেখান থেকে throughput আসে — বিনিময়ে read amplification-এর খরচ, যা Bloom filter কমায়, এবং compaction, যার জন্য আমি headroom provision করতাম।" এই উত্তরটি দেখায় যে আপনি একটি database সম্পর্কে যুক্তি দিতে পারেন, শুধু এর marketing মুখস্থ করেননি। এটি এটাও ব্যাখ্যা করে **কেন** Bloom filter system design-এ বারবার দেখা যায়, যা topic 42-কে স্বেচ্ছাচারী নয়, বরং অনিবার্য মনে করায়।

## এটি কীভাবে সংযুক্ত

এটি **indexing** (topic 12)-এর নিচের স্তর এবং **SQL vs NoSQL** (topic 11)-এর পেছনের performance profile-এর গভীর কারণ ব্যাখ্যা করে। **Bloom filter** (topic 42) LSM read-এর একটি core component। **MVCC** (topic 37) এই structure-গুলোর উপর ভিত্তি করে implement করা হয়। Write throughput বৈশিষ্ট্য সরাসরি **sharding** সিদ্ধান্তে (topic 14) প্রভাব ফেলে, এবং compaction-চালিত latency variance এমন কিছু যা **observability** (topic 43)-কে আশ্চর্যচকিত করার আগেই প্রকাশ করতে হয়।

**পরবর্তী:** [GraphQL: A Query-Based Alternative to REST](../39-graphql-a-query-based-alternative-to-rest/why.md) — API layer-এ ফিরে যাওয়া, এবং data চাওয়ার একটি ভিন্ন উপায়।
</content>
