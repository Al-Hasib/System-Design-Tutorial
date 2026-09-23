# LSM Trees vs. B-Trees: Storage Engine Internals

**কঠিনতার মাত্রা:** Advanced

## শেখার লক্ষ্য (Learning Objectives)

- B-tree index কীভাবে কাজ করে এবং ঐতিহ্যবাহী relational database-গুলোর access pattern-এর জন্য এটি কেন optimize করা, তা স্মরণ করা।
- একটি LSM (Log-Structured Merge) tree আসলে কী এবং এটি কীভাবে random write-কে sequential write-এ রূপান্তর করে, তা ব্যাখ্যা করা।
- compaction কী এবং LSM tree-র দীর্ঘমেয়াদী সুস্থতার জন্য কেন এটি প্রয়োজনীয়, তা বর্ণনা করা।
- B-tree এবং LSM tree-র মধ্যে read, write, এবং space amplification trade-off তুলনা করা।
- একটি নির্দিষ্ট workload-এর read/write ratio অনুযায়ী সঠিক storage engine family বেছে নেওয়া।

## স্ক্রিপ্ট (Script)

### শুরু / ভূমিকা

Module 3-এ আমরা indexing নিয়ে একটি conceptual আলোচনা করেছিলাম: range query-র জন্য B-tree, exact lookup-এর জন্য hash index। সেটা সত্যি, কিন্তু এটি নীরবে ধরে নেয় যে প্রতিটি database B-tree ব্যবহার করে — অথচ system design interview-তে বাস্তবে যেসব database নিয়ে আপনি design করবেন, তার একটি বিশাল অংশ — Cassandra থেকে শুরু করে RocksDB, এমনকি অনেক time-series ও logging system-এর নিচে থাকা engine পর্যন্ত — তা করে না। এগুলো বরং LSM tree নামে কিছু একটা ব্যবহার করে। আজ আমরা দেখব: B-tree আসলে কীসের জন্য optimize করা, LSM tree structurally এমন কোন সমস্যার সমাধান করে যা B-tree ভালোভাবে করতে পারে না, এবং আপনার workload আসলে কোনটি চায় তা কীভাবে যুক্তি দিয়ে বোঝা যায়।

### B-Tree: In-Place Update-এর জন্য Optimized

একটি B-tree index data-কে fixed-size page-এর একটি balanced tree-তে সাজায় (প্রায়ই disk-এর block size-এর সাথে মিলিয়ে), যেখানে প্রতিটি page-এ sorted key এবং child page-এর pointer থাকে। কোনো row খুঁজে পেতে হলে আপনাকে tree-র নিচের দিকে হাঁটতে হয় — মুষ্টিমেয় কয়েকটি page read-ই যেকোনো row-এ পৌঁছে দেয়, যে কারণে B-tree চমৎকার, predictable read performance দেয়, range scan সহ, কারণ sorted key-গুলো disk-এ কাছাকাছি বসে থাকে।

খরচটা দেখা যায় write-এর সময়। কোনো row insert বা update করলে, B-tree-কে সঠিক page খুঁজে বের করে সেটিকে *in place* modify করতে হয় — এবং সেই page disk-এর যেকোনো জায়গায় থাকতে পারে। বড় scale-এ, যখন অনেক ভিন্ন key জুড়ে write ছড়িয়ে থাকে, এর মানে হলো ধারাবাহিক random disk I/O: এই page-এ seek করো, লেখো, তারপর সম্পূর্ণ ভিন্ন একটা page-এ seek করো, আবার লেখো। Random I/O sequential I/O-র তুলনায় অনেক বেশি ধীর, বিশেষত spinning disk-এ, এবং SSD-তেও এটি sequential write-এর চেয়ে বেশি খরচসাপেক্ষ, কারণ flash storage ভেতরে ভেতরে ছোট random write-কে কীভাবে সামলায়। B-tree-কে মাঝেমধ্যে একটি পূর্ণ page-কে দুটিতে split করে tree rebalance করতেও হয় — write path-এ আরও overhead। এই কারণেই B-tree-ভিত্তিক database (PostgreSQL, MySQL/InnoDB, বেশিরভাগ ঐতিহ্যবাহী relational database) read-heavy এবং range-query-heavy workload-এ চমৎকার, কিন্তু অত্যন্ত উচ্চ, sustained write volume-এর নিচে চাপে পড়তে শুরু করে।

### LSM Tree: Random Write-কে Sequential-এ রূপান্তর

একটি Log-Structured Merge tree সম্পূর্ণ ভিন্ন একটি পদ্ধতি নেয়, যা একটি মূল ধারণার উপর ভিত্তি করে তৈরি: **কখনোই data-কে in place modify করা হয় না — শুধু append করা হয়**। কোনো write এলে, প্রথমে সেটি একটি in-memory structure-এ লেখা হয় (সাধারণত যাকে বলা হয় **memtable**, প্রায়ই skip list-এর মতো sorted structure দিয়ে backed), এবং crash-এর ক্ষেত্রে durability নিশ্চিত করতে একটি on-disk write-ahead log-এও লেখা হয়। এই দুটোই sequential append operation — সঠিক page খুঁজতে disk-এ কোথাও seek করার দরকার নেই। memtable পূর্ণ হয়ে গেলে, সেটি disk-এ flush করা হয় একটি immutable, sorted file হিসেবে (যাকে প্রায়ই বলা হয় **SSTable** — Sorted String Table)। সময়ের সাথে সাথে, disk-এ এমন অনেকগুলো immutable SSTable জমা হয়।

এই design write-কে অত্যন্ত দ্রুত করে তোলে, কারণ প্রতিটি write শুধুই একটি append — sequential I/O, কোনো page-splitting নেই, কোনো in-place seek-and-modify নেই। trade-off দেখা যায় read-এর সময়: যেহেতু একই key memtable-এ এবং একাধিক ভিন্ন SSTable-এ থাকতে পারে (একটি update পুরনো ভার্সনকে overwrite না করে শুধু একটি নতুন ভার্সন append করে), একটি read-কে একাধিক জায়গা check করতে হতে পারে এবং যা পাওয়া যায় তার মধ্যে সবচেয়ে নতুন ভার্সনটি ফেরত দিতে হয়। Database-গুলো এটি কমাতে in-memory Bloom filter ব্যবহার করে (যদি filter বলে key নিশ্চিতভাবে নেই, তাহলে একটি read সম্পূর্ণভাবে একটি SSTable এড়িয়ে যেতে পারে) এবং পর্যায়ক্রমে **compaction** চালিয়ে — একটি background process যা একাধিক SSTable-কে একত্র করে merge করে, updated বা deleted key-র পুরনো, প্রতিস্থাপিত ভার্সন বাদ দিয়ে, এবং কম সংখ্যক, বড়, তখনও sorted file তৈরি করে। compaction-ই read performance এবং disk usage-কে আরও বেশি SSTable জমা হওয়ার সাথে সাথে অনির্দিষ্টকালের জন্য খারাপ হতে দেয় না — এটি চলমান background কাজ, যার জন্য LSM-tree database-গুলোকে CPU এবং I/O বরাদ্দ রাখতে হয়।

### Trade-off, স্পষ্টভাবে নামকরণ

এটি তিন ধরনের "amplification"-এ এসে দাঁড়ায়, যেগুলোর শব্দভাণ্ডার interview-এর জন্য মুখস্থ রাখা উচিত: **Write amplification** — অ্যাপ্লিকেশন যে লজিক্যাল byte লিখেছে তার জন্য disk-এ আসলে কতগুলো byte লেখা হয় (B-tree-তে page rewrite ও split-এর কারণে write amplification বেশি হতে পারে; LSM tree-তে write-path amplification কম, কিন্তু পরে compaction-এর সময় সেটা পরিশোধ করতে হয়)। **Read amplification** — একটি logical read-এর উত্তর দিতে কতগুলো disk read দরকার (B-tree: সাধারণত কম এবং predictable, মাত্র কয়েকটি page read; LSM tree: সম্ভাব্য একাধিক SSTable check করতে হয়, যা Bloom filter এবং compaction দিয়ে কমানো হয়)। **Space amplification** — logical data size-এর চেয়ে অতিরিক্ত কতটা disk space ব্যবহার হয় (LSM tree সাময়িকভাবে updated/deleted data-র একাধিক ভার্সন সংরক্ষণ করে যতক্ষণ না compaction সেটা পরিষ্কার করে; B-tree-তে সাধারণত এই সমস্যা হয় না কারণ update in place ঘটে)। কোনো storage engine একসাথে তিনটিকেই minimize করতে পারে না — প্রতিটি LSM-tree এবং B-tree implementation এই trade-off triangle-এর একটি নির্দিষ্ট বিন্দু, যা তার target workload-এর জন্য tune করা।

### বাস্তব-জগতের উদাহরণ

একটি ঐতিহ্যবাহী relational database, যা একটি e-commerce product catalog-এর পেছনে কাজ করছে (মাঝারি write volume, range query এবং join-এ ভারী — PostgreSQL-এর মতো B-tree engine এখানে স্বাভাবিকভাবে মানানসই), বনাম একটি time-series metrics store যা হাজার হাজার সার্ভার থেকে প্রতি সেকেন্ডে লক্ষ লক্ষ data point ingest করছে (অত্যন্ত উচ্চ, প্রায় সম্পূর্ণ-append write volume, যেখানে বেশিরভাগ query সাম্প্রতিক data-র জন্য — Cassandra বা InfluxDB-র underlying storage-এর মতো LSM-tree engine এখানে স্বাভাবিকভাবে মানানসই) — এই দুটোর পার্থক্য নিয়ে ভাবুন। অথবা সরাসরি Cassandra-র কথা ভাবুন: এটি সম্পূর্ণরূপে LSM tree-র উপর ভিত্তি করে তৈরি, ঠিক এই কারণে যে এর design লক্ষ্য হলো একটি distributed cluster জুড়ে বিশাল write throughput শোষণ করা, এবং এটি সেই write performance-এর মূল্য হিসেবে read-side জটিলতা (একাধিক SSTable, Bloom filter, compaction) মেনে নেয় — যা ঠিক সেই write-heavy, append-heavy workload-এর সাথে মিলে যায় যার জন্য এটি সাধারণত বেছে নেওয়া হয়।

### সংক্ষিপ্তকরণ (Recap)

B-tree data-কে in place update করে, যা চমৎকার, predictable read এবং range-query performance দেয়, কিন্তু বিনিময়ে random-write I/O এবং page-splitting overhead-এর খরচ বহন করতে হয় — read-heavy, মাঝারি-write relational workload-এর জন্য এটি সঠিক পছন্দ। LSM tree কখনোই data-কে in place modify করে না — এটি শুধু append করে, write-কে দ্রুত sequential I/O-তে রূপান্তরিত করে — কিন্তু বিনিময়ে read-এর সময় একাধিক file check করতে হয় (যা Bloom filter দিয়ে কমানো হয়) এবং read performance ও disk usage নিয়ন্ত্রণে রাখতে চলমান background compaction প্রয়োজন হয়। "একটি database" মানে "একটি B-tree" — এমনটি ধরে নেওয়ার আগে আপনার workload-এর প্রকৃত read/write ratio এবং access pattern নিয়ে যুক্তি দিন — আধুনিক NoSQL ও time-series system-গুলোর একটি বিশাল অংশ ঠিক এই কারণেই LSM tree-র উপর নির্মিত।

### এরপর কী

আমরা এখন database internals-এর দুটি স্তম্ভ নিয়ে গভীরভাবে আলোচনা করেছি: কীভাবে transaction isolated থাকে, এবং নিচে থাকা storage engine আসলে কীভাবে data persist করে। এবার আমরা গিয়ার পরিবর্তন করি — পরবর্তী video-তে আমরা Module 2 থেকে স্থগিত রাখা একটি API architecture প্রশ্ন দেখব: GraphQL structurally আসলে কী সমাধান করে যা REST করতে পারে না, এবং কোথায় এটি তার যোগ করা জটিলতার মূল্য অর্জন করে।

## মূল বিষয়বস্তু (Key Takeaways)

- B-tree data-কে in place update করে, যা দ্রুত, predictable read এবং range scan দেয়, কিন্তু বিনিময়ে write-এ random-write I/O এবং page-splitting overhead-এর খরচ বহন করতে হয়।
- LSM tree কখনোই data-কে in place modify করে না — write হলো memtable এবং write-ahead log-এ sequential append, যা পরে disk-এ immutable, sorted SSTable হিসেবে flush হয়।
- LSM tree read-path জটিলতা (একাধিক SSTable check করা, যা Bloom filter দিয়ে কমানো হয়) এবং চলমান background compaction-এর বিনিময়ে অনেক দ্রুত sustained write throughput দেয়।
- Write, read, এবং space amplification — এই তিনটি axis-এর মধ্যে প্রতিটি storage engine-কে trade-off করতে হয় — কোনো engine একসাথে তিনটিকেই minimize করতে পারে না।
- workload অনুযায়ী বেছে নিন: read-heavy/range-query-heavy হলে B-tree উপযুক্ত (বেশিরভাগ relational database); write-heavy/append-heavy হলে LSM tree উপযুক্ত (Cassandra, RocksDB, বেশিরভাগ time-series store)।
</content>
