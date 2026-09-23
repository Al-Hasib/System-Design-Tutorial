# Database Replication: Master-Slave & Master-Master

**কঠিনতার মাত্রা:** Intermediate

## শেখার লক্ষ্যসমূহ (Learning Objectives)

- database replication আসলে কী এবং কেন একটি একক database instance একইসাথে reliability-এর ঝুঁকি ও scaling-এর বাধা, তা ব্যাখ্যা করা।
- synchronous এবং asynchronous replication-এর তুলনা করা এবং তাদের মধ্যকার durability/consistency বনাম latency-র trade-off স্পষ্ট করা।
- Master-Slave (leader-follower) replication কীভাবে কাজ করে তা বর্ণনা করা, যার মধ্যে রয়েছে read scaling, replication lag, এবং failover।
- Master-Master (multi-leader) replication কীভাবে কাজ করে তা বর্ণনা করা, যার মধ্যে রয়েছে write conflict শনাক্তকরণ ও সমাধানের কৌশল।
- read/write প্যাটার্ন, consistency-র প্রয়োজন এবং ভৌগোলিক বিস্তারের ভিত্তিতে একটি নির্দিষ্ট পরিস্থিতির জন্য কোন replication topology উপযুক্ত তা মূল্যায়ন করা।

## স্ক্রিপ্ট (Script)

### Hook/Intro

কল্পনা করুন: আপনি একটি অ্যাপ বানিয়েছেন, এর একটিমাত্র database server আছে, আর সেটি বেশ ভালোভাবেই চলছে। তারপর একদিন, সেই server-এর disk নষ্ট হয়ে যায়। অথবা হয়তো নষ্ট হয় না — শুধু আর সামলাতে পারে না, কারণ আপনার ব্যবহারকারী সংখ্যা হাজার থেকে মিলিয়নে পৌঁছে গেছে, আর প্রতিটি read এবং write সেই একটিমাত্র মেশিনে গিয়ে পড়ছে। যেভাবেই হোক, আপনার একটি সমস্যা আছে, আর উভয় ক্ষেত্রেই এর মূল কারণ একই: আপনার একটি single point of failure এবং একটি single point of scale রয়েছে। একটি মেশিন বন্ধ হয়ে গেলে সাথে আপনার পুরো অ্যাপও বন্ধ হয়ে যায়, আর একটি মেশিনের ক্ষমতা নির্ধারণ করে দেয় আপনি কতটুকু ট্রাফিক সামলাতে পারবেন তার একটি কঠোর সীমা।

আজ আমরা এই সমস্যাটি সমাধান করব **database replication**-এর মাধ্যমে — এক database node থেকে ডেটা কপি করে এক বা একাধিক অন্য node-এ রাখার অনুশীলন, যাতে একটি মাত্র মেশিনের বদলে একই ডেটা একাধিক মেশিনে থাকে। Replication হলো distributed systems-এর অন্যতম মৌলিক কৌশল, এবং আপনি যত production database ব্যবহার করেছেন তার প্রায় সবগুলোর ভেতরেই এটি বিদ্যমান — PostgreSQL, MySQL, MongoDB, সবগুলোই এটি native ভাবে সমর্থন করে। আমরা এখানে দেখব replication আসলে কী, synchronous ও asynchronous replication-এর মধ্যে পার্থক্য কী, এবং তারপর গভীরভাবে দেখব সেই দুটি বড় topology, যেগুলো নিয়ে যেকোনো system design interview-তে আপনাকে জিজ্ঞাসা করা হবে: Master-Slave এবং Master-Master।

### Replication আসলে কী

মূলগতভাবে, replication মানে হলো: আপনার ডেটার একাধিক কপি একাধিক মেশিনে রাখা, এবং সেই কপিগুলোকে সিঙ্কে রাখা। কেন এত ঝামেলা করা হয়? দুটি বড় কারণ আছে। প্রথমত, **availability এবং durability** — যদি একটি node মারা যায়, অন্য একটি node-এ ইতিমধ্যেই ডেটা থাকে এবং সেটি দায়িত্ব নিতে পারে, ফলে আপনি ডেটা হারান না এবং সার্ভিসও বন্ধ হয় না। দ্বিতীয়ত, **read scaling** — যদি একাধিক মেশিনে ডেটার একাধিক কপি থাকে, তাহলে আপনি read ট্রাফিক একটিমাত্র server-এ চাপ না দিয়ে সবগুলোর মধ্যে ছড়িয়ে দিতে পারেন।

কিন্তু replication বিনামূল্যে পাওয়া যায় না। যখনই একই ডেটার একাধিক কপি থাকবে, তখনই আপনাকে একটি কঠিন প্রশ্নের উত্তর দিতে হবে: আপনি কীভাবে সেগুলোকে সিঙ্কে রাখবেন, এবং যখন তারা সাময়িকভাবে একমত না হয় তখন কী ঘটে? এখানেই synchronous বনাম asynchronous replication-এর প্রসঙ্গ আসে।

### Synchronous বনাম Asynchronous Replication

**Synchronous replication**-এ, যখন একজন client ডেটা লেখে, primary node সেই write-টি replica-তে পাঠায়, এবং client-কে "আপনার write সফল হয়েছে" বলার আগে replica থেকে confirmation-এর জন্য অপেক্ষা করে যে সে write-টি পেয়েছে এবং প্রয়োগ করেছে। এটিকে একটি certified letter পাঠানোর মতো ভাবুন — যতক্ষণ না signed receipt ফিরে পাচ্ছেন, ততক্ষণ কাজ শেষ হয়েছে বলে মনে করেন না। এর সুবিধা হলো শক্তিশালী durability এবং consistency: যদি primary write acknowledge করার ঠিক পরমুহূর্তেই মারা যায়, তাহলে আপনি জানেন যে replica-তেও ইতিমধ্যে সেই ডেটা আছে, তাই কিছুই হারায়নি। অসুবিধা হলো latency — এখন প্রতিটি write-কে replica-তে একটি network round trip-এর জন্য অপেক্ষা করতে হয়, যা ধীরগতির, এবং যদি replica অপ্রাপ্য হয়, তাহলে আপনার write সম্পূর্ণভাবে থমকে যেতে পারে।

**Asynchronous replication**-এ, primary client-কে সাথে সাথেই write acknowledge করে দেয়, এবং তারপর ব্যাকগ্রাউন্ডে, যখন সুযোগ পায়, replica-গুলোতে update পাঠায়। এটি একটি সাধারণ চিঠি ডাকবাক্সে ফেলে দেওয়ার মতো — receipt-এর জন্য অপেক্ষা না করেই আপনি আপনার দিনের কাজে চলে যান। Write দ্রুত হয় কারণ আপনাকে replica-তে network round trip-এর জন্য অপেক্ষা করতে হয় না। কিন্তু এখন এমন একটি সময়কাল থাকে যখন primary-র কাছে এমন ডেটা থাকে যা replica-গুলোর কাছে এখনো নেই। যদি সেই সময়কালে primary ক্র্যাশ করে, তাহলে সেই ডেটা হারিয়ে যেতে পারে। "primary-র কাছে আছে" এবং "replica-র কাছে আছে"-এর মধ্যকার এই ফাঁককে বলা হয় **replication lag**, এবং এটি এই পুরো বিষয়ের অন্যতম গুরুত্বপূর্ণ ধারণা — আমরা বারবার এর কাছে ফিরে আসব।

বাস্তব জগতের বেশিরভাগ সিস্টেম পারফরম্যান্সের জন্য ডিফল্টভাবে asynchronous replication ব্যবহার করে, এবং নির্বাচিতভাবে synchronous replication ব্যবহার করে — উদাহরণস্বরূপ, শুধুমাত্র একটি replica-কে গুরুত্বপূর্ণ write synchronously acknowledge করতে বলা, বাকিগুলো asynchronously replicate হয়। এটি এমন একটি trade-off যা আপনি কতটা durability দরকার তার বিপরীতে কতটা latency সহ্য করতে পারেন তার ভিত্তিতে টিউন করেন।

### Master-Slave (Leader-Follower) Replication

সবচেয়ে সাধারণ replication topology হলো **Master-Slave**, যাকে ক্রমশ **leader-follower** replication-ও বলা হচ্ছে, কারণ "master/slave" পরিভাষাটি ইন্ডাস্ট্রি জুড়ে ধীরে ধীরে বাদ দিয়ে "primary/replica" বা "leader/follower" ব্যবহার করা হচ্ছে — তবে আপনি এখনো অনেক ডকুমেন্টেশন ও interview প্রশ্নে master-slave দেখতে পাবেন, তাই দুটি নামই জানা ভালো।

এটি এভাবে কাজ করে: আপনি একটি node-কে **master** (বা leader) হিসেবে নির্ধারণ করেন, এবং বাকি সব node হলো **slave** (বা follower, বা read replica)। সমস্ত write — insert, update, delete — একচেটিয়াভাবে master-এ যায়। এরপর master সেই পরিবর্তনগুলো প্রতিটি follower-এ স্ট্রিম করে, সাধারণত asynchronously। অন্যদিকে, read *যেকোনো* node-এ যেতে পারে — master বা যেকোনো follower-এ। এটিই বড় সুবিধা: যেহেতু বেশিরভাগ অ্যাপ্লিকেশন read-heavy, তাই আপনি আরও follower replica যোগ করেই প্রায় লিনিয়ারভাবে আপনার read ক্ষমতা বাড়াতে পারেন, আপনার write ক্ষমতা বা ডেটা মডেল স্পর্শ না করেই।

কিন্তু এতে আমাদের পরিচিত replication lag চলে আসে। যেহেতু follower-এ replication সাধারণত asynchronous, তাই master-এ একটি write পড়া এবং একটি follower-এ সেটি দেখা যাওয়ার মধ্যে একটি ছোট বিলম্ব থাকে — প্রায়ই মিলিসেকেন্ড, তবে ভারী লোডে কখনো কখনো সেকেন্ডও। এটি একটি চিরায়ত এবং খুবই বাস্তব বাগ তৈরি করে, যাকে বলা হয় **read-your-writes problem**: একজন ব্যবহারকারী একটি মন্তব্য পোস্ট করে, write master-এ যায়, অ্যাপ সাথে সাথেই তাকে এমন একটি পাতায় নিয়ে যায় যা একটি follower থেকে পড়ে, এবং সেই follower তখনও সেই write পায়নি — ফলে ব্যবহারকারী দেখেন তার নিজের মন্তব্যটিই অনুপস্থিত। চিরায়ত সমাধানগুলোর মধ্যে রয়েছে write করার পর অল্প কিছু সময়ের জন্য ব্যবহারকারীর নিজের read master-এ পাঠানো, replication position-এর সাথে সংযুক্ত একটি "read your own writes" token ট্র্যাক করা, অথবা সহজভাবে session-critical read master-এ পাঠানো।

এখন, master নিজেই মারা গেলে কী হয়? একে বলা হয় **failover**, এবং এটি master-slave replication-এর সবচেয়ে জটিল অপারেশনাল অংশ। সিস্টেমকে — হয় একটি automated tool বা একজন human operator — সনাক্ত করতে হয় যে master বন্ধ হয়ে গেছে, follower-দের মধ্যে থেকে একটিকে নতুন master হিসেবে promote করার জন্য বেছে নিতে হয়, এবং ভবিষ্যতের সব write সেখানে পুনর্নির্দেশিত করতে হয়। এতে সময় লাগে, যে সময়ে সাধারণত write অনুপলব্ধ থাকে, এবং যদি asynchronous replication ব্যবহার করা হয়ে থাকে, তাহলে promote করা follower-এ যেসব write তখনো replicate হয়নি সেগুলো হারানোর ঝুঁকি থাকে। PostgreSQL-এর Patroni বা MySQL Group Replication-এর মতো টুলগুলো এর অনেকটা automate করে, কিন্তু এটি কখনোই তাৎক্ষণিক বা ঝুঁকিমুক্ত নয়।

### Master-Master (Multi-Leader) Replication

**Master-Master**, বা **multi-leader replication**, ভিন্ন একটি পদ্ধতি নেয়: একটি মাত্র master সব write গ্রহণ করার বদলে, *একাধিক* node সরাসরি write গ্রহণ করতে পারে, এবং তারা একে অপরের কাছে পরিবর্তনগুলো replicate করে। এটি এমন একটি সমস্যা সমাধান করে যা master-slave করতে পারে না: একাধিক লোকেশন জুড়ে write scaling এবং write availability। যদি আপনার US এবং Europe-এ ব্যবহারকারী থাকে, আপনি প্রতিটি অঞ্চলে একটি করে master চালাতে পারেন, এবং ব্যবহারকারীরা ভৌগোলিকভাবে যে master তাদের সবচেয়ে কাছে সেখানে write করে — অনেক কম write latency, এবং যদি একটি অঞ্চলের master বন্ধ হয়ে যায়, অন্যটি কোনো failover প্রক্রিয়া ছাড়াই write গ্রহণ চালিয়ে যায়।

কিন্তু multi-leader replication একটি সত্যিকারের কঠিন সমস্যা খুলে দেয়: **write conflicts**। যদি একজন US ব্যবহারকারী একটি রেকর্ড আপডেট করে ঠিক একই মুহূর্তে যখন একজন Europe ব্যবহারকারী *একই* রেকর্ড আপডেট করে, তাহলে উভয় master-ই স্থানীয়ভাবে write গ্রহণ করে, এবং তারপর যখন তারা একে অপরের কাছে replicate করে, তখন একই ডেটার জন্য দুটি ভিন্ন মান সঠিক বলে দাবি করে। কাউকে না কাউকে সিদ্ধান্ত নিতে হয় কে জিতবে।

সবচেয়ে সহজ কৌশল হলো **last-write-wins (LWW)**: প্রতিটি write-এর সাথে একটি timestamp যুক্ত করা, এবং যখন একটি conflict সনাক্ত করা হয়, তখন যে write-এর timestamp পরে সেটিই টিকে থাকে। এটি সহজ এবং অনেক সিস্টেম ডিফল্টভাবে এটিই ব্যবহার করে, কিন্তু এটি একটি স্থূল হাতিয়ারও বটে — এটি নীরবে একটি বৈধ write বাতিল করে দিতে পারে শুধুমাত্র এই কারণে যে clock-গুলো সামান্য অসিঙ্ক্রোনাস, অথবা দুটি write খুব কাছাকাছি সময়ে ঘটেছিল। আরও পরিশীলিত সিস্টেমগুলো **vector clocks** ব্যবহার করে — এমন একটি data structure যা node জুড়ে update-এর কার্যকারণ (causal) ইতিহাস ট্র্যাক করে, যাতে সিস্টেম বুঝতে পারে একটি write আসলে অন্যটির *পরে ঘটেছে* কিনা (এবং সেটি ওভাররাইট করা উচিত কিনা), নাকি সেগুলো সত্যিকার অর্থেই একযোগে ঘটেছে এবং merging-এর জন্য ফ্ল্যাগ করা দরকার, কখনো কখনো এমনকি সমাধানের জন্য অ্যাপ্লিকেশন বা ব্যবহারকারীর কাছে ফিরিয়ে দেওয়া হয়। CouchDB-এর মতো কিছু database নীরবে বিজয়ী বেছে নেওয়ার পরিবর্তে সরাসরি conflict অ্যাপ্লিকেশন layer-এ প্রকাশ করে দেয়।

Master-master **multi-region active-active** আর্কিটেকচারে সবচেয়ে ভালোভাবে কাজ করে, যেখানে low latency এবং আঞ্চলিক fault tolerance-এর জন্য সত্যিই প্রতিটি অঞ্চলের read এবং write উভয়ই গ্রহণ করা প্রয়োজন — যেমন globally distributed collaboration tool বা shopping cart। কিন্তু এই নমনীয়তার জন্য conflict resolution-এর চারপাশে প্রকৃত অপারেশনাল ও অ্যাপ্লিকেশন জটিলতার মূল্য দিতে হয়।

### দুটি Topology-র তুলনা

চলুন এগুলোকে সরাসরি পাশাপাশি রাখি। **write scaling**-এর ক্ষেত্রে, master-slave সীমিত — সব write একটি node দিয়ে যায় — যেখানে master-master একাধিক node এবং অঞ্চল জুড়ে write scale করতে দেয়। **read scaling**-এর ক্ষেত্রে, উভয়ই ভালোভাবে সামলায়, কারণ উভয়ই read ট্রাফিক শোষণ করার জন্য follower বা অতিরিক্ত leader যোগ করতে দেয়। **conflict risk**-এর ক্ষেত্রে, master-slave-এ মূলত কোনো ঝুঁকিই নেই, কারণ সেখানে সবসময় একজনই writer থাকে; master-master-এ প্রকৃত conflict risk আছে যা আপনাকে সক্রিয়ভাবে ডিজাইন করতে হবে। **complexity**-র ক্ষেত্রে, master-slave বোঝা ও পরিচালনা করা সহজ; master-master conflict resolution এবং বহুমুখী replication-এর কারণে উল্লেখযোগ্যভাবে বেশি জটিল। **failover**-এর ক্ষেত্রে, master-slave-এর জন্য কিছুটা downtime বা data-loss ঝুঁকিসহ একটি স্পষ্ট promotion প্রক্রিয়া দরকার; master-master-এ write-এর জন্য কোনো single point of failure নেই, কারণ একটি master বন্ধ হয়ে গেলেও অন্য master-গুলো write গ্রহণ চালিয়ে যায়। সাধারণ নিয়ম হলো: ডিফল্টভাবে master-slave বেছে নিন, কারণ এটি সহজ এবং বেশিরভাগ read-heavy workload কভার করে, এবং শুধুমাত্র তখনই master-master বেছে নিন যখন আপনার multi-region write availability-র একটি নির্দিষ্ট প্রয়োজন থাকে যা অতিরিক্ত জটিলতাকে যুক্তিসঙ্গত করে তোলে।

### বাস্তব-জগতের উদাহরণ

একটি read-heavy ব্লগিং প্ল্যাটফর্মের কথা বিবেচনা করুন: প্রতিটি একটি write-এর (একটি নতুন পোস্ট বা মন্তব্য) বিপরীতে হয়তো হাজারটি read (মানুষ কনটেন্ট ব্রাউজ ও দেখছে) থাকতে পারে। একটি আদর্শ master-slave সেটআপ এটি চমৎকারভাবে সামলায় — একটি master তুলনামূলকভাবে বিরল write সামলায়, এবং আপনি বিশাল read ভলিউম শোষণ করার জন্য, ধরুন, একটি load balancer-এর পেছনে পাঁচটি read replica স্থাপন করেন, ট্রাফিক বাড়ার সাথে সাথে শুধু আরও replica যোগ করে read ক্ষমতা বাড়ান। এখন এর সাথে তুলনা করুন North America, Europe এবং Asia-তে সক্রিয় ব্যবহারকারীসহ একটি বৈশ্বিক collaborative document editor-এর, যেখানে সবার low latency-সহ দ্রুত write দরকার এবং একটি অঞ্চলে outage হলেও availability চালু থাকা দরকার। এটি একটি আদর্শ master-master case: প্রতি অঞ্চলে একটি master, asynchronous cross-region replication, এবং একটি conflict resolution কৌশল — প্রায়ই vector clocks বা operational-transform-স্টাইল merging — একযোগে সম্পাদিত edit-গুলো মিলিয়ে নেওয়ার জন্য।

### সংক্ষিপ্তসার (Recap)

চলুন সংক্ষিপ্ত করি। Replication মানে হলো availability এবং read scaling-এর জন্য একাধিক node জুড়ে আপনার ডেটার একাধিক কপি রাখা। Synchronous replication replica confirmation-এর জন্য অপেক্ষা করে durability-র বিনিময়ে latency দেয়; asynchronous replication গতির বিনিময়ে সামান্য durability ঝুঁকি নেয় এবং replication lag তৈরি করে। Master-slave replication সব write একটি master-এ পাঠায় এবং follower-দের মধ্যে read বিতরণ করে — সহজ, কিন্তু write scaling-এ সীমিত এবং master মারা গেলে একটি failover প্রক্রিয়া দরকার হয়। Master-master replication একাধিক node-কে write গ্রহণ করতে দেয়, write throughput বাড়ায় এবং multi-region active-active সেটআপ সক্ষম করে, তবে এর জন্য last-write-wins বা vector clocks-এর মতো একটি conflict resolution কৌশল দরকার।

### এরপর কী

Replication *read* scale করার জন্য চমৎকার — আপনি read ট্রাফিকের জন্য যত খুশি replica যোগ করতে পারেন। কিন্তু লক্ষ্য করুন এটি কী সমাধান করে *না*: এটি আপনাকে একটি মেশিন (বা, master-master-এ, প্রতিটি সম্পূর্ণ ডেটাসেট ধারণকারী কয়েকটি মেশিন) যা সামলাতে পারে তার চেয়ে বেশি *write* scale করতে সাহায্য করে না, এবং যখন আপনার মোট ডেটাসেট একটি মেশিনের ডিস্কে ধরার জন্য একেবারেই খুব বড় হয়ে যায়, তখনও এটি সাহায্য করে না। এর জন্য, আপনার প্রয়োজন একটি মৌলিকভাবে ভিন্ন কৌশল: সবকিছু সর্বত্র কপি করার বদলে, আপনার ডেটাকেই node জুড়ে ভাগ করা। এটিই ঠিক যা আমরা পরবর্তী ভিডিওতে কভার করছি: **Database Sharding & Partitioning Strategies**। সেখানে দেখা হচ্ছে।

## মূল শিক্ষণীয় বিষয় (Key Takeaways)

- একটি একক database instance একইসাথে একটি single point of failure এবং scale-এর একটি কঠোর সীমা — replication node জুড়ে ডেটার একাধিক কপি বজায় রেখে উভয়টিই সমাধান করে।
- Synchronous replication write latency-র বিনিময়ে durability/consistency-কে অগ্রাধিকার দেয়; asynchronous replication এমন একটি replication lag window-এর বিনিময়ে গতিকে অগ্রাধিকার দেয় যেখানে primary ব্যর্থ হলে ডেটা হারাতে পারে।
- Master-Slave (leader-follower) replication সব write একটি master-এ পাঠায় এবং follower-দের মধ্যে read বিতরণ করে, যা শক্তিশালী read scaling দেয় কিন্তু একটি failover প্রক্রিয়া দরকার হয় এবং replication lag-এর কারণে সৃষ্ট read-your-writes problem-এর শিকার হয়।
- Master-Master (multi-leader) replication একাধিক node-এ write গ্রহণ করে, write scaling এবং multi-region active-active আর্কিটেকচার সক্ষম করে, কিন্তু এর জন্য last-write-wins বা vector clocks-এর মতো একটি conflict resolution কৌশল দরকার।
- Replication read scale করে (এবং, master-master-এর ক্ষেত্রে, কিছুটা write-ও), কিন্তু মোট ডেটাসেট আকার বা মৌলিক write-throughput সীমা সমাধান করে না — সেটি sharding-এর কাজ, যা পরবর্তীতে কভার করা হয়েছে।
