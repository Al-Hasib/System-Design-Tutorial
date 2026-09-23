# Database Sharding & Partitioning Strategies

**কঠিনতার মাত্রা:** Intermediate/Advanced

## শেখার লক্ষ্যসমূহ

এই ভিডিও শেষে আপনি নিচের বিষয়গুলো বুঝতে পারবেন:

- sharding এবং partitioning বলতে কী বোঝায়, এবং এগুলো replication থেকে কীভাবে আলাদা তা ব্যাখ্যা করা।
- vertical (functional) partitioning-কে horizontal partitioning (sharding) থেকে আলাদা করা।
- range-based, hash-based, এবং directory-based sharding strategy-গুলোর মধ্যে তুলনা করা, তাদের trade-off সহ।
- একটি ভালো shard key কীসে তৈরি হয় তা মূল্যায়ন করা এবং সাধারণ hotspot ফাঁদ এড়ানো।
- sharding যেসব operational চ্যালেঞ্জ নিয়ে আসে, যেমন cross-shard join এবং rebalancing, তা চিহ্নিত করা।

## স্ক্রিপ্ট

### শুরু/ভূমিকা

আগের ভিডিওতে আমরা replication নিয়ে কথা বলেছিলাম — আপনার ডেটা একাধিক node-এ কপি করে রাখা, যাতে failure থেকে বেঁচে থাকতে পারেন এবং read traffic ছড়িয়ে দিতে পারেন। এটা একটা বাস্তব সমস্যার সমাধান করে। আপনার অ্যাপ যদি read-heavy হয়, তাহলে replication দিয়ে আপনি read replica যোগ করে চালিয়ে যেতে পারেন। কিন্তু replication যেটা সমাধান করে *না*, তা হলো: write scaling, এবং raw dataset-এর আকার।

একটু ভাবুন। একটি master-slave সেটআপে, প্রতিটি write-কে এখনও একটি মাত্র primary node দিয়ে যেতে হয়। আপনার দশটি read replica থাকুক বা একশোটি, তাতে কিছু যায় আসে না — write সবসময় সেই একটি মেশিনের CPU, memory, এবং disk I/O-তে bottleneck হয়ে থাকবে। এমনকি আপনি যদি শুধু read-ই করেন, একটা সময় আপনার dataset-ই এত বড় হয়ে যাবে যে এক মেশিনে ধরবে না, বা এতটাই বড় হয়ে যাবে যে দক্ষভাবে query করা যাবে না—যদিও টেকনিক্যালি ধরে। Replication প্রতিটি node-কে একই ডেটার *সম্পূর্ণ কপি* দেয়। এটা availability-র জন্য দারুণ, কিন্তু প্রতিটি node-কে যে পরিমাণ ডেটা সামলাতে হয় তা কমায় না।

তাহলে যখন একটি ডেটাবেস সার্ভার সহজেই আপনার সমস্ত ডেটা এবং সমস্ত write ধরে রাখতে বা সামলাতে পারে না, তখন কী করবেন? আপনি সেটাকে ভাগ করে ফেলবেন। এটাই sharding — এবং আজকের ভিডিওর বিষয় এটাই।

### Sharding / Partitioning আসলে কী?

আসুন শব্দগুলো সংজ্ঞায়িত করি, কারণ মানুষ "partitioning" এবং "sharding" শব্দ দুটো একটু আলগাভাবে ব্যবহার করে।

বিস্তৃত অর্থে, Partitioning মানে হলো আপনার ডেটাকে ছোট ছোট, বেশি manageable অংশে ভাগ করা। Sharding হলো partitioning-এর একটি নির্দিষ্ট *ধরন*: একাধিক আলাদা ডেটাবেস instance-এ ডেটা ভাগ করা — ভিন্ন ভিন্ন সার্ভার, সম্ভবত ভিন্ন rack বা region-এ — যেখানে প্রতিটি instance, যাকে shard বলা হয়, মোট ডেটার শুধু একটি অংশ ধরে রাখে।

এখানে একটা analogy দিই। কল্পনা করুন এক কোটি বই সহ একটি বিশাল লাইব্রেরি, সব একটিমাত্র ভবনে গাদাগাদি করে রাখা। প্রতিটি দর্শনার্থী, প্রতিটি লাইব্রেরিয়ান, প্রতিটি ডেলিভারি ট্রাক সেই একটি ভবনের একটি মাত্র প্রবেশপথ দিয়ে যায়। শেষপর্যন্ত ভবনটি পূর্ণ হয়ে যায়, আর করিডোরগুলো জ্যামে ভরে যায়। Sharding হলো একটির বদলে দশটি শাখা লাইব্রেরি খোলার মতো, প্রতিটিতে বইয়ের এক-দশমাংশ থাকবে। এখন, একটির বদলে দশটি প্রধান দরজা ট্রাফিক সামলায়, এবং প্রতিটি শাখাকে শুধু তার নিজের অংশের সংগ্রহ সংরক্ষণ ও index করতে হয়।

### Vertical বনাম Horizontal Partitioning

আরও এগোনোর আগে, আসুন দুটো ধারণাকে আলাদা করি যেগুলোকে উভয়কেই "partitioning" বলা হয়।

**Vertical partitioning** আপনার ডেটাকে *column* বা *feature/table* অনুযায়ী ভাগ করে। উদাহরণস্বরূপ, আপনি হয়তো আপনার `users` টেবিলটি একটি ডেটাবেসে রাখবেন এবং `orders` টেবিলটি সম্পূর্ণ ভিন্ন একটি ডেটাবেসে রাখবেন, কারণ এগুলো ভিন্ন ভিন্ন সার্ভিস দ্বারা অ্যাক্সেস করা হয়। অথবা একটি একক টেবিলের মধ্যেই, আপনি হয়তো কম ব্যবহৃত, বড় column-গুলো — যেমন একজন ব্যবহারকারীর বায়োগ্রাফি বা প্রোফাইল ছবির blob — আলাদা একটি টেবিলে ভাগ করবেন, যাতে "hot" column-গুলো ছোট থাকে এবং দ্রুত scan করা যায়। এটা মূলত আপনার schema-কে বিভিন্ন data store জুড়ে ফাংশনাল ভাবে ভাগ করা।

**Horizontal partitioning**, যাকে আমরা sharding বলি, ডেটাকে *row* অনুযায়ী ভাগ করে। আপনি একটি single logical table নেন — ধরুন, `users` — এবং এর row-গুলো একাধিক ডেটাবেস instance জুড়ে ভাগ করেন। Shard 1-এ হয়তো ১ থেকে ১ কোটি ব্যবহারকারী থাকবে, shard 2-এ পরবর্তী ১ কোটি, এভাবে চলতে থাকে। প্রতিটি shard-এর schema হুবহু একই থাকে; শুধু ভিন্ন ভিন্ন row থাকে। এই টেকনিকটাই আসলে storage এবং write throughput উভয়কেই horizontally scale করতে দেয়, কারণ প্রতিটি shard একটি স্বাধীন ডেটাবেস যা শুধু নিজের row-এর অংশ নিয়েই কাজ করে।

আজ আমরা মূলত horizontal partitioning — sharding — নিয়ে ফোকাস করব, কারণ এটাই সেই অংশ যা read এবং write উভয়ের জন্যই প্রকৃত horizontal scale সম্ভব করে।

### Sharding Strategies

তো আপনি যখন row-গুলোকে shard জুড়ে ভাগ করার সিদ্ধান্ত নিলেন, তখন কীভাবে ঠিক করবেন *কোন row কোন shard-এ যাবে*? এর তিনটি classic strategy আছে।

**Range-based sharding।** আপনি একটি shard key বেছে নেন — ধরুন, `user_id` — এবং সেই key-এর contiguous range প্রতিটি shard-কে বরাদ্দ করেন। ১ থেকে ১০ লাখ ব্যবহারকারী shard A-তে যায়, ১০ লাখ ১ থেকে ২০ লাখ shard B-তে যায়, এভাবে চলতে থাকে। এর বড় সুবিধা হলো range query সস্তা এবং স্বাভাবিক — "আমাকে ID ৫ লাখ থেকে ৬ লাখের মধ্যে সব ব্যবহারকারী দাও" ঠিক একটি shard-এ গিয়ে পড়ে। সমস্যাটা হলো hotspot। যদি আপনার key কোনো signup timestamp বা auto-incrementing ID-এর মতো কিছু হয়, তাহলে আপনার *সবচেয়ে নতুন* — এবং প্রায়শই সবচেয়ে active — ডেটা *সর্বশেষ* shard-এ গিয়ে পড়ে, যখন আপনার পুরনো shard-গুলো তুলনামূলক অলস বসে থাকে। সংখ্যার হিসাবে "সমান" ডেটা বণ্টন থাকলেও আপনি অসম load পান।

**Hash-based sharding।** এখানে, আপনি shard key-কে একটি hash function-এর মধ্য দিয়ে চালান, এবং hash-এর ফলাফল — সাধারণত shard-সংখ্যা দিয়ে modulo করে — নির্ধারণ করে row কোন shard-এ থাকবে। যেহেতু একটি ভালো hash function key-গুলোকে pseudo-randomly ছড়িয়ে দেয়, এটি data volume এবং load উভয়কেই shard জুড়ে খুব সমানভাবে ছড়িয়ে দিতে থাকে — এক range-এ hotspot জমা হওয়ার আর সুযোগ নেই। এর trade-off হলো আপনি দক্ষভাবে range query করার ক্ষমতা হারান। "মার্চ থেকে এপ্রিলের মধ্যে সব order আমাকে দাও" আর একটি shard-এর সাথে map করে না; আপনাকে হয়তো প্রতিটি shard query করে ফলাফল merge করতে হবে, কারণ যেসব row মূল key-তে sequential ছিল সেগুলো এখন random ভাবে ছড়িয়ে গেছে।

**Directory-based sharding।** কোনো formula দিয়ে ডেটা কোথায় থাকবে তা হিসাব করার বদলে, আপনি একটি explicit lookup service — একটি directory — বজায় রাখেন যা প্রতিটি key, বা key-এর প্রতিটি range-কে একটি নির্দিষ্ট shard-এর সাথে map করে। জানতে চান user 42 কোথায় থাকে? Directory service-কে জিজ্ঞেস করুন, সেটা বলে দেবে "shard 3"। এটা এখন পর্যন্ত সবচেয়ে *flexible* পদ্ধতি: আপনি individual key-গুলো shard-এর মধ্যে সরাতে পারেন, অসমভাবে loaded shard-গুলোকে rebalance করতে পারেন, এবং কোনো কঠোর mathematical formula অনুসরণ না করেই capacity যোগ করতে পারেন। এর খরচ হলো বাড়তি complexity এবং প্রতিটি query-তে একটি অতিরিক্ত network hop — সেই সাথে directory service নিজেই একটি critical infrastructure হয়ে ওঠে যাকে দ্রুত, available, এবং consistent থাকতে হয়।

এখন, range এবং সাধারণ hash-modulo sharding উভয়েরই একটি সাধারণ সমস্যা হলো resharding-এর কষ্ট। যদি আপনি naive hash-modulo — hash mod N — দিয়ে একটি shard যোগ বা বাদ দেন, তাহলে N পরিবর্তন প্রায় প্রতিটি key-এর target shard পুনর্বিন্যাস করে দেয়, মানে আপনাকে প্রায় সব ডেটা সরাতে হবে। **Consistent hashing** নামে একটি টেকনিক আছে যা shard-সংখ্যা বাড়ানো বা কমানোর সময় কতটা ডেটা সরাতে হবে তা নাটকীয়ভাবে কমিয়ে দেয়। আমি এখানে এটা নিয়ে গভীরে যাচ্ছি না — এটা diagram সহ নিজস্ব একটি ফোকাসড ব্যাখ্যা পাওয়ার যোগ্য — কিন্তু জেনে রাখুন এটা আছে, এটা বাস্তব distributed system-এ ব্যাপকভাবে ব্যবহৃত হয়, এবং আমরা পরবর্তী কোনো ভিডিওতে এটা বিস্তারিত কভার করব।

### Shard Key নির্বাচন করা

সঠিক shard key বেছে নেওয়া সম্ভবত একটি sharding design-এ সবচেয়ে গুরুত্বপূর্ণ সিদ্ধান্ত, কারণ পরে এটা পরিবর্তন করা কষ্টকর। কয়েকটা বিষয় খেয়াল রাখতে হবে:

প্রথমত, **cardinality** — আপনি এমন একটি key চান যাতে যথেষ্ট distinct value থাকে যাতে ডেটা আসলেই আপনার সব shard জুড়ে ছড়িয়ে পড়ে। `country` দিয়ে sharding করা যুক্তিসঙ্গত মনে হতে পারে, কিন্তু যদি আপনার ৮০% ব্যবহারকারী একটি দেশে থাকে, তাহলে আপনি যেভাবেই ভাগ করুন না কেন সেই shard-টা hotspot হয়ে যাবে।

দ্বিতীয়ত, **access pattern**। আপনার সবচেয়ে সাধারণ query-গুলো দেখুন। যদি প্রায় প্রতিটি query `tenant_id` বা `customer_id` দিয়ে filter করে, তাহলে সেই key দিয়ে sharding করার মানে হলো বেশিরভাগ query-কে fan out এবং সব shard জুড়ে ফলাফল merge না করেই একটি মাত্র shard-এ route করা যায় — এটা performance এবং সরলতার দিক থেকে বিশাল লাভ।

তৃতীয়ত, **hotspot এড়ানো**। সময়ের সাথে সম্পর্কিত বা sequential ID-এর মতো key, celebrity/power-user প্রভাব যেখানে একটি key অন্যদের তুলনায় ব্যাপকভাবে বেশি traffic পায়, এবং সাধারণভাবে বাস্তব জগতের skewed distribution থেকে সতর্ক থাকুন।

### Sharding-এর চ্যালেঞ্জ

Sharding বিনামূল্যে নয় — এটা প্রকৃত operational complexity নিয়ে আসে।

**Cross-shard join এবং transaction** কঠিন হয়ে যায়। যদি সম্পর্কিত ডেটা ভিন্ন ভিন্ন shard-এ থাকে, তাহলে join মানে এখন একাধিক ডেটাবেস query করে application code-এ ফলাফল combine করা, এবং দুটি shard জুড়ে বিস্তৃত একটি transaction-এর জন্য একটি সাধারণ local ACID transaction-এর বদলে distributed transaction coordination দরকার হয় — যেমন two-phase commit বা saga pattern।

**Rebalancing** কষ্টকর। ডেটা যখন অসমভাবে বাড়ে, তখন আপনাকে শেষপর্যন্ত shard-এর মধ্যে ডেটা সরাতে হবে বা নতুন shard যোগ করতে হবে, এবং আপনার strategy-র উপর নির্ভর করে, এর মানে হতে পারে সর্বনিম্ন downtime নিয়ে বিশাল পরিমাণ ডেটা migrate করা — সত্যিকারের একটি কঠিন operational সমস্যা।

এবং সামগ্রিকভাবে **operational complexity** যথেষ্ট বেড়ে যায়। একটি ডেটাবেস পরিচালনার বদলে, আপনি N-টি ডেটাবেস পরিচালনা করছেন, প্রতিটির monitoring, backup, সামঞ্জস্যপূর্ণভাবে প্রয়োগ করা schema migration, এবং capacity planning দরকার — যত shard চালাচ্ছেন তার সংখ্যা দিয়ে গুণ করা।

### বাস্তব উদাহরণ

আসুন এটাকে concrete করি। ধরুন আপনি একটি e-commerce প্ল্যাটফর্ম তৈরি করছেন, এবং আপনার `orders` টেবিল একটি single Postgres instance যা comfortably সামলাতে পারে তার চেয়ে বড় হয়ে গেছে।

একটি পদ্ধতি: `customer_id` দিয়ে shard করুন, hash-based sharding ব্যবহার করে, ধরুন, ১৬টি shard জুড়ে। একজন customer-এর প্রতিটি order একই shard-এ hash হয়, তাই "এই customer-এর সব order দাও" — আপনার সবচেয়ে সাধারণ query-গুলোর একটি — সবসময় ঠিক একটি shard-এ গিয়ে পড়ে। নতুন customer এবং তাদের order volume সমানভাবে বণ্টন হয় কারণ hash function signup date বা region-এর তোয়াক্কা করে না।

বিকল্পভাবে, কল্পনা করুন একটি global users table যা hash ব্যবহার করে `user_id` দিয়ে shard করা হয়েছে। একজন ব্যবহারকারী যখন লগ ইন করে, আপনার application তাদের ID hash করে, দেখে আপনার কোন shard সেই hash range-এর মালিক, এবং সরাসরি সেখানে query route করে। User ID দিয়ে সাধারণ point lookup দ্রুত এবং single-shard থাকে; শুধু cross-user analytical query — যেমন "গত সপ্তাহে পুরো প্ল্যাটফর্ম জুড়ে কতজন ব্যবহারকারী সাইন আপ করেছে" — এখন প্রতিটি shard-এ fan out করে আপনার application layer-এ aggregate করতে হয়।

### সংক্ষিপ্তসার

আসুন এটাকে একসাথে বাঁধি। Replication একই ডেটা সবখানে কপি করে availability এবং read scaling-এ সাহায্য করে। Sharding — horizontal partitioning — write এবং মোট storage scale করতে *ভিন্ন* row ভিন্ন ভিন্ন ডেটাবেস instance জুড়ে ভাগ করে। বিপরীতে, Vertical partitioning আপনার schema-কে row অনুযায়ী নয়, table বা column অনুযায়ী ভাগ করে। Strategy-র ক্ষেত্রে, range-based sharding আপনাকে সস্তা range query দেয় কিন্তু hotspot-এর ঝুঁকি রাখে; hash-based sharding সমান বণ্টন দেয় কিন্তু range query ত্যাগ করে; directory-based sharding একটি অতিরিক্ত lookup hop এবং বাড়তি infrastructure-এর বিনিময়ে সর্বোচ্চ flexibility দেয়। Consistent hashing এসবের সাথে আসা resharding-এর কষ্ট কমাতে সাহায্য করে। এবং আপনার shard key নির্বাচন — cardinality, access pattern, এবং hotspot এড়ানো দ্বারা চালিত — এমন একটি সিদ্ধান্ত যা পুরো design-কে সফল বা ব্যর্থ করে দেবে।

### এরপর কী

তো এখন আমরা distributed-data toolbox-এর দুটি বড় হাতিয়ার কভার করেছি: কপির জন্য replication, এবং ভাগের জন্য sharding। এগুলো একত্রিত করুন — এবং আপনি এমন সিস্টেম পাবেন যা horizontally scaled *এবং* fault-tolerant উভয়ই। কিন্তু সেই সংমিশ্রণ বিনামূল্যে আসে না। প্রতিবার আপনি replicate বা shard করার সময়, আপনি consistency, availability, এবং latency-র মধ্যে implicit trade-off করছেন। পরবর্তী ভিডিওতে, আমরা CAP theorem এবং এর আরও সূক্ষ্ম আত্মীয় PACELC দিয়ে সেই trade-off-গুলোকে explicit করব — যাতে আপনি ঠিক বুঝতে পারেন প্রতিবার আপনার ডেটা distribute করার সময় আপনি কী ছেড়ে দিচ্ছেন, এবং কী পাচ্ছেন।

## মূল শিক্ষণীয় বিষয়

- Sharding (horizontal partitioning) একটি table-এর row-গুলো একাধিক ডেটাবেস instance জুড়ে ভাগ করে write এবং মোট storage scale করে — যা একা replication করতে পারে না।
- Vertical partitioning ডেটাকে table/column অনুযায়ী ভাগ করে; horizontal partitioning (sharding) ডেটাকে row অনুযায়ী আলাদা instance জুড়ে ভাগ করে।
- Range-based sharding দক্ষ range query সমর্থন করে কিন্তু sequential বা time-correlated key-তে hotspot-এর ঝুঁকি রাখে।
- Hash-based sharding load সমানভাবে বণ্টন করে কিন্তু range query ব্যয়বহুল করে তোলে, কারণ এর জন্য প্রতিটি shard-এ fan out করতে হয়।
- Directory-based sharding একটি lookup service-এর মাধ্যমে সবচেয়ে বেশি flexibility দেয়, একটি অতিরিক্ত network hop এবং বাড়তি infrastructure-এর বিনিময়ে।
- Consistent hashing shard যোগ/বাদ দেওয়ার সময় প্রয়োজনীয় ডেটা movement-এর পরিমাণ কমায় (পরে বিস্তারিত কভার করা হবে)।
- একটি ভালো shard key-তে থাকে উচ্চ cardinality, আপনার প্রধান access pattern-এর সাথে মিল, এবং অল্প সংখ্যক value-তে load কেন্দ্রীভূত হওয়া এড়ানো।
- Sharding প্রকৃত operational cost নিয়ে আসে: cross-shard join/transaction, rebalancing, এবং বহুগুণ operational overhead।
