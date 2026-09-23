# ACID vs BASE, Normalization vs Denormalization

**Difficulty:** Intermediate

## Learning Objectives

এই ভিডিও শেষে আপনি নিম্নলিখিত বিষয়গুলো করতে পারবেন:

- ACID-এর প্রতিটি অক্ষর এবং BASE-এর প্রতিটি শব্দ ব্যাখ্যা করতে পারবেন, প্রতিটির জন্য একটি করে বাস্তব উদাহরণসহ।
- ACID এবং BASE-কে CAP theorem এবং PACELC trade-off-এর সাথে সংযুক্ত করতে পারবেন।
- চারটি প্রমিত transaction isolation level কনসেপ্চুয়াল লেভেলে বর্ণনা করতে পারবেন।
- normalization (1NF/2NF/3NF) এবং denormalization ব্যাখ্যা করতে পারবেন, এবং এই দুইয়ের মধ্যকার trade-off বুঝতে পারবেন।
- একটি নির্দিষ্ট পরিস্থিতির জন্য সিদ্ধান্ত নিতে পারবেন যে normalized/ACID ডিজাইন নাকি denormalized/BASE ডিজাইন বেশি উপযুক্ত।

## Script

### Hook/Intro

সবাইকে স্বাগতম, আবার ফিরে এসেছি। আগের ভিডিওতে আমরা CAP theorem এবং PACELC নিয়ে কথা বলেছিলাম — এই ধারণা যে যখন একটি network partition ঘটে, তখন একটি distributed system-কে হয় consistent থাকা, নয়তো available থাকার মধ্যে একটি বেছে নিতে হয়, এবং partition না থাকলেও latency আর consistency-র মধ্যে একটি trade-off থেকে যায়।

আজ আমরা সেই trade-off-এর consistency দিকটায় জুম করব এবং প্রশ্ন করব: database লেভেলে "consistency" আসলে কেমন অনুভূত হয়? সেখানেই আসে দুটি acronym: ACID এবং BASE। এখানে আমরা যা আগে থেকে জানি তার সাথে সংযোগ আছে — CP-leaning সিস্টেম, যারা availability-র চেয়ে consistency-কে প্রাধান্য দেয়, সাধারণত ACID guarantee দেয়। যেমন ধরুন PostgreSQL বা MySQL-এর মতো traditional relational database, যা single-node বা strongly consistent configuration-এ চলছে। AP-leaning সিস্টেম, যারা availability-কে প্রাধান্য দেয়, সাধারণত এর বদলে BASE guarantee দেয়। যেমন Cassandra, DynamoDB, বা একটি globally replicated document store। কোনোটাই "ভালো" নয় — এগুলো একই মূল প্রশ্নের ভিন্ন ভিন্ন উত্তর: correctness নিশ্চিত করতে একটি সিস্টেম কতটুকু ধীর হতে বা "না" বলতে রাজি।

আর একবার এটা বুঝে গেলে, আমরা একটি ঘনিষ্ঠভাবে সম্পর্কিত কিন্তু আলাদা বিষয়ে যাব: আপনি আসলে কীভাবে আপনার data shape করবেন — normalized নাকি denormalized — কারণ এই সিদ্ধান্তটি গভীরভাবে জড়িত আপনি কোন consistency model-এর অধীনে কাজ করছেন তার সাথে।

### ACID, বিস্তারিতভাবে

চলুন ACID দিয়ে শুরু করি। এটি দাঁড়িয়েছে Atomicity, Consistency, Isolation, এবং Durability-এর জন্য, এবং এটি transaction-এর জন্য relational database-গুলো যে classic guarantee দেয় তা।

**Atomicity** মানে একটি transaction হয় সম্পূর্ণভাবে ঘটবে, নয়তো একেবারেই ঘটবে না। যদি আপনি Alice-এর অ্যাকাউন্ট থেকে Bob-এর অ্যাকাউন্টে $100 ট্রান্সফার করেন, তাতে দুটি write জড়িত থাকে: Alice-কে debit করা, Bob-কে credit করা। Atomicity নিশ্চিত করে যে হয় দুটি write-ই ঘটবে, নয়তো কোনোটিই না। যদি Alice-কে debit করার ঠিক পরে কিন্তু Bob-কে credit করার আগে সিস্টেম ক্র্যাশ করে, তাহলে database পুরো transaction rollback করে দেয়। কখনোই এমন হবে না যে টাকা বাতাসে মিলিয়ে গেল।

**Consistency** — এবং লক্ষ্য করুন এটি CAP-এর "C" থেকে আলাদা, যা অনেককে বিভ্রান্ত করে — মানে একটি transaction database-কে এক valid state থেকে আরেক valid state-এ নিয়ে যায়, সব সংজ্ঞায়িত নিয়ম মেনে: constraints, foreign keys, triggers, cascades। যদি আপনার schema বলে যে অ্যাকাউন্ট ব্যালেন্স কখনো ঋণাত্মক হতে পারবে না, তাহলে consistency নিশ্চিত করে যে কোনো committed transaction কখনোই সেই নিয়ম ভাঙবে না।

**Isolation** মানে সমান্তরাল (concurrent) transaction-গুলো একে অপরের সাথে হস্তক্ষেপ করে না। যদি দুইজন ব্যক্তি একই মুহূর্তে একটি ফ্লাইটের শেষ সিটটি বুক করার চেষ্টা করেন, isolation নিশ্চিত করে যে database correctness-এর দৃষ্টিকোণ থেকে তাদের একটার পর একটা হ্যান্ডেল করে — একই সিট দুইবার বিক্রি হবে না, যদিও দুটি request একসাথে (parallel-এ) এসেছিল।

**Durability** মানে একবার একটি transaction commit হয়ে গেলে, তা টিকে থাকে — এমনকি তার ঠিক পরেই power failure বা crash হলেও। Database সাধারণত commit স্বীকার করার আগে ডিস্কে একটি transaction log-এ লিখে রাখে, যাতে data শুধু মেমোরিতে বসে হারিয়ে না যায়।

সবকিছু একসাথে করলে, ACID হলো এমন কিছু যা আপনাকে টাকা, ইনভেন্টরি সংখ্যা, বা এমন যেকোনো কিছুর জন্য database-কে বিশ্বাস করার সুযোগ দেয় যেখানে "মোটামুটি সঠিক" যথেষ্ট নয়।

### Isolation Level নিয়ে সংক্ষিপ্ত আলোচনা

যেহেতু isolation আসলে একটি স্পেকট্রাম, একটি একক আচরণ নয়, তাই SQL standard-এ সংজ্ঞায়িত চারটি প্রমিত isolation level জানা গুরুত্বপূর্ণ, এমনকি শুধু উচ্চ-স্তরেও। **Read Uncommitted** সবচেয়ে শিথিল — আপনি অন্য transaction-এর uncommitted পরিবর্তন দেখতে পারেন, তথাকথিত "dirty reads"। **Read Committed** — PostgreSQL-এর মতো অনেক database-এ ডিফল্ট — শুধু committed data দেখতে দেয়, কিন্তু একই transaction-এর মধ্যে বারবার read করলে ভিন্ন মান আসতে পারে যদি অন্য একটি transaction মাঝখানে commit করে। **Repeatable Read** সেটা ঠিক করে দেয়: একটি transaction-এর ভেতরে, একই query সবসময় একই row রিটার্ন করে। আর **Serializable** সবচেয়ে কঠোর — transaction-গুলো এমনভাবে আচরণ করে যেন তারা একে একে চলেছে, সম্পূর্ণভাবে isolated, যদিও তারা আসলে সমান্তরালে চলছে। isolation যত টাইট, আপনি তত নিরাপদ, কিন্তু সাধারণত তত বেশি contention এবং তত কম throughput। আপাতত বিস্তারিত মুখস্থ করার দরকার নেই — শুধু জানুন যে এই ডায়াল বিদ্যমান এবং এটি ACID-এর মধ্যেই একটি trade-off নব।

### BASE, বিস্তারিতভাবে

এখন BASE-এ চলে যাই, যা অনেক distributed এবং NoSQL সিস্টেম এর পরিবর্তে গ্রহণ করে। এটি দাঁড়িয়েছে Basically Available, Soft state, এবং Eventual consistency-এর জন্য।

**Basically Available** মানে সিস্টেম প্রতিটি request-এ সাড়া দেওয়াকে প্রাধান্য দেয়, এমনকি যদি সেই সাড়া একদম সর্বশেষ write প্রতিফলিত করার নিশ্চয়তা না দেয়। একটি network hiccup-এর সময় block বা error করার বদলে, এটি gracefully degrade করে এবং তবুও একটি উত্তর দেয়।

**Soft state** মানে সিস্টেমের state সময়ের সাথে পরিবর্তিত হতে পারে এমনকি নতুন কোনো ইনপুট ছাড়াই, শুধুমাত্র ব্যাকগ্রাউন্ডে replication ক্যাচ-আপ করার কারণে। ACID-এর isolation-এর মতো data লকড থাকে না — এটি প্রবাহমান থাকে যখন পরিবর্তনগুলো নোডগুলোতে ছড়িয়ে পড়ে।

**Eventual consistency** হলো সবচেয়ে বড় বিষয়: যদি আপনি কোনো data-তে write করা বন্ধ করে দেন, সব replica অবশেষে একই মানে converge করবে — কিন্তু ঠিক কখন তা হবে তার কোনো নিশ্চয়তা নেই। একটি write অন্য একটি নোডে পৌঁছানোর ঠিক পরেই আপনি এক replica থেকে সামান্য stale data পড়তে পারেন।

এটাকে সরাসরি ACID-এর সাথে তুলনা করুন: যেখানে ACID বলে "আমি তোমাকে অপেক্ষা করাব যতক্ষণ না আমি correctness নিশ্চিত করতে পারি," BASE বলে "আমি সবসময় তোমাকে উত্তর দেব, আর correctness শীঘ্রই ধরে ফেলবে।" এটাই ঠিক CAP theorem-এর AP দিকটি database ডিজাইনে দেখা যাওয়া। Distributed NoSQL সিস্টেমগুলো BASE-কে প্রাধান্য দেয় কারণ অনেকগুলো ভৌগোলিকভাবে ছড়িয়ে থাকা নোড জুড়ে কঠোর ACID-স্টাইল isolation এবং consistency প্রয়োগ করা ব্যয়বহুল — এর জন্য coordination দরকার, আর coordination-এর খরচ হলো latency এবং availability। যদি আপনি "like" counter বা news feed-এর মতো কিছু বানাচ্ছেন, তাহলে কয়েক সেকেন্ডের staleness এমন একটি সিস্টেমের জন্য সম্পূর্ণ গ্রহণযোগ্য মূল্য যা কখনো ডাউন হয় না এবং সবসময় দ্রুত সাড়া দেয়।

### কখন ACID বনাম BASE বেছে নেবেন

তাহলে কীভাবে বেছে নেবেন? জিজ্ঞাসা করুন data ভুল বা stale হলে কী হয়, এমনকি সংক্ষিপ্ত সময়ের জন্যও। financial ledger, inventory system, বা টাকা বা unique constraint জড়িত এমন যেকোনো কিছুর জন্য — যেমন username বা seat assignment — ACID বেছে নিন। একটি ভুলের খরচ বেশি, এবং raw availability-র চেয়ে correctness বেশি গুরুত্বপূর্ণ। social media feed, analytics dashboard, product view counter, বা activity log-এর মতো জিনিসের জন্য — BASE বেছে নিন। কয়েক সেকেন্ডের staleness-এর খরচ কম, এবং আপনি বরং বিশ্বজুড়ে অসাধারণ গতি এবং 24/7 availability পেতে চাইবেন।

### Normalization vs Denormalization

এখন একটি সম্পর্কিত কিন্তু আলাদা সিদ্ধান্তে চলে যাই: আপনি কীভাবে আপনার data shape করবেন।

**Normalization** হলো relational database-এর সেই শৃঙ্খলা যা redundancy দূর করার জন্য data সংগঠিত করে। কনসেপ্চুয়াল লেভেলে: **First Normal Form (1NF)** বলে যে প্রতিটি column একটি একক, atomic মান ধারণ করবে — একটি field-এ comma দিয়ে আলাদা করা তালিকা গুঁজে দেওয়া যাবে না। **Second Normal Form (2NF)** বলে যে প্রতিটি non-key column-কে পুরো primary key-এর উপর নির্ভরশীল হতে হবে, শুধু তার একটি অংশের উপর নয় — এটা composite key যুক্ত table-এর ক্ষেত্রে গুরুত্বপূর্ণ। **Third Normal Form (3NF)** বলে যে প্রতিটি non-key column শুধুমাত্র key-এর উপর নির্ভরশীল হবে, অন্য কোনো non-key column-এর উপর নয়। বাস্তবে, normalization মানে data-কে আলাদা আলাদা table-এ ভাগ করা — Users, Orders, Products — যেগুলো foreign key দিয়ে সংযুক্ত থাকে, যাতে প্রতিটি তথ্য ঠিক একবার সংরক্ষিত থাকে। এটি "update anomalies" এড়ায়: কল্পনা করুন একজন গ্রাহকের ঠিকানা এক হাজার অর্ডার row-এ পুনরাবৃত্তভাবে সংরক্ষিত আছে। তারা যদি স্থানান্তরিত হন, আপনাকে হয় এক হাজার row আপডেট করতে হবে, নয়তো অসামঞ্জস্যপূর্ণ ঠিকানা থেকে যাবে। Normalization এটা ঠিক করে দেয় ঠিকানাটি একবার, Users table-এ সংরক্ষণ করে।

**Denormalization** হলো এর ইচ্ছাকৃত বিপরীত: read-এর সময় join-এর প্রয়োজন এড়াতে রেকর্ড জুড়ে data নকল করা। যদি আপনার social media অ্যাপকে instantly একটি feed রেন্ডার করতে হয়, তাহলে আপনি প্রতিটি post document-এর ভেতরে সরাসরি লেখকের নাম এবং avatar-এর একটি কপি সংরক্ষণ করতে পারেন, প্রতিটি read-এ Users collection-এর সাথে join করার বদলে। এটি NoSQL এবং analytics সিস্টেমে অত্যন্ত সাধারণ, যেখানে বড় স্কেলে read speed storage efficiency বা একটি single source of truth-এর চেয়ে বেশি গুরুত্বপূর্ণ।

### Trade-off গুলো

Normalization আপনাকে একটি single source of truth, কম storage, এবং নিরাপদ write দেয় — কিন্তু সম্পর্কিত data একত্রিত করা প্রয়োজন এমন read-এর জন্য join লাগে, যা বড় স্কেলে ব্যয়বহুল হয়ে উঠতে পারে। Denormalization join ছাড়াই দ্রুত read দেয়, যা উচ্চ-ট্রাফিক, read-heavy workload-এর জন্য দারুণ — কিন্তু write আরও জটিল হয়ে যায়, কারণ একটি তথ্য আপডেট করার মানে হতে পারে অনেকগুলো নকল কপি স্পর্শ করা, এবং আপনি সেই কপিগুলোর একে অপরের সাথে সিঙ্ক থেকে বিচ্যুত হওয়ার একটি বাস্তব ঝুঁকি তৈরি করেন। এটি classic space-versus-speed এবং write-simplicity-versus-read-simplicity trade-off, এবং — লক্ষ্য করুন — এটি প্রায় নিখুঁতভাবে ACID versus BASE-এর সাথে মিলে যায়। Strongly consistent, normalized schema-গুলো সাধারণত ACID transactional database-এর সাথে হাত ধরাধরি করে চলে। Denormalized, duplicated data সাধারণত BASE, eventually consistent সিস্টেমের সাথে হাত ধরাধরি করে চলে, কারণ একটি distributed system জুড়ে real time-এ নকল কপিগুলোকে নিখুঁতভাবে সিঙ্কে রাখা ঠিক সেই ধরনের coordination যা BASE সিস্টেমগুলো এড়াতে চায়।

### বাস্তব-বিশ্বের উদাহরণ

চলুন এটা মাটিতে নামিয়ে আনি। একটি ব্যাংকের core ledger সিস্টেম একটি textbook ACID এবং normalized use case। অ্যাকাউন্ট ব্যালেন্স, transaction history, এবং গ্রাহকের রেকর্ড কঠোর foreign key সহ একটি normalized relational schema-তে থাকে, এবং প্রতিটি ট্রান্সফার একটি ACID transaction-এর ভেতরে চলে — atomic, isolated, durable — কারণ একটি একক হারিয়ে যাওয়া বা দুইবার প্রয়োগ হওয়া transaction একটি গুরুতর সমস্যা, আর্থিক এবং আইনগত উভয় দিক থেকেই।

এখন সেটাকে একটি social media feed-এর সাথে তুলনা করুন। আপনি যখন অ্যাপ খোলেন, আপনি চান আপনার feed মিলিসেকেন্ডে লোড হোক, শত শত মিলিয়ন ব্যবহারকারীকে ক্রমাগত তৈরি হতে থাকা content দেখাক। সেই সিস্টেম আক্রমণাত্মকভাবে denormalize করে — প্রতিটি post লেখকের প্রোফাইল তথ্যের একটি নকল স্ন্যাপশট বহন করতে পারে, like সংখ্যা আনুমানিক এবং asynchronously আপডেট হয়, এবং বিভিন্ন ব্যবহারকারী কয়েক সেকেন্ডের জন্য like counter-এর সামান্য ভিন্ন, সামান্য stale সংস্করণ দেখতে পারেন। এটাই BASE কার্যকরভাবে: basically available, eventually consistent, নিখুঁত real-time নির্ভুলতার চেয়ে read speed এবং uptime-এর জন্য অপ্টিমাইজড।

### Recap

চলুন সংক্ষেপে দেখি। ACID — Atomicity, Consistency, Isolation, Durability — আপনাকে শক্তিশালী, বিশ্বস্ত transactional guarantee দেয়, সাধারণত distributed অবস্থায় availability এবং latency-র খরচে। BASE — Basically Available, Soft state, Eventual consistency — কঠোর correctness-কে availability এবং speed-এর বিনিময়ে ছেড়ে দেয়, যে কারণে distributed এবং NoSQL সিস্টেমগুলো এটা পছন্দ করে। আলাদাভাবে, normalization redundancy দূর করতে এবং write correctness রক্ষা করতে data সংগঠিত করে, আর denormalization write complexity এবং consistency risk-এর বিনিময়ে read দ্রুততর করতে data নকল করে। আর এই দুটি সিদ্ধান্ত সাধারণত একসাথে চলে: ACID স্বাভাবিকভাবে normalized schema-র সাথে জোড় বাঁধে, আর BASE স্বাভাবিকভাবে denormalized schema-র সাথে জোড় বাঁধে।

### এরপর কী

এখন যেহেতু আমরা বুঝেছি আমরা কীভাবে data সংরক্ষণ করি তার মধ্যে বেক করা consistency trade-off গুলো, এরপর আমরা যাচ্ছি Module 4-এ, শুরু হচ্ছে "Caching Strategies and Cache Invalidation" দিয়ে। একবার আপনি এই storage সিস্টেমগুলোর যেকোনো একটির সামনে একটি cache রাখলে, আপনি আপনার data-এর আরেকটি কপি চালু করেন — এবং আরেকটি consistency চ্যালেঞ্জ: আপনি কীভাবে জানবেন সেই cache করা কপিটি কখন stale, এবং সে সম্পর্কে আপনি কী করবেন? সেখানে দেখা হবে।

## Key Takeaways

- ACID (Atomicity, Consistency, Isolation, Durability) শক্তিশালী transactional guarantee প্রদান করে এবং financial ledger-এর মতো CP-leaning, single-source-of-truth সিস্টেমের জন্য স্বাভাবিক পছন্দ।
- BASE (Basically Available, Soft state, Eventual consistency) কঠোর correctness-কে availability এবং speed-এর বিনিময়ে ছাড় দেয়, এবং AP-leaning, distributed NoSQL সিস্টেমের জন্য স্বাভাবিক পছন্দ।
- Isolation level (read uncommitted, read committed, repeatable read, serializable) ACID-এর ভেতরেই একটি টিউনযোগ্য স্পেকট্রাম, যা correctness-কে concurrency এবং throughput-এর বিনিময়ে দেয়।
- Normalization redundancy কমায় এবং update anomaly প্রতিরোধ করে কিন্তু join প্রয়োজন; denormalization দ্রুত read-এর জন্য data নকল করে কিন্তু write complexity এবং consistency risk যোগ করে।
- ACID/normalization এবং BASE/denormalization সাধারণত স্বাভাবিকভাবে জোড় বাঁধে, উভয়ই CAP এবং PACELC থেকে আসা একই অন্তর্নিহিত consistency-vs-availability/speed trade-off প্রতিফলিত করে।
