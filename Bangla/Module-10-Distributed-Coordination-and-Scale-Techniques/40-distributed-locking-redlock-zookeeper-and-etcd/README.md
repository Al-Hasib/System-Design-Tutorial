# Distributed Locking: Redlock, ZooKeeper & etcd

**কঠিনতা:** Advanced

## শিখনের লক্ষ্যসমূহ (Learning Objectives)

- ব্যাখ্যা করা কেন একটি সাধারণ in-process lock (একটি mutex) একাধিক machine জুড়ে access coordinate করতে পারে না।
- Redlock algorithm এবং এটি যে নির্দিষ্ট race condition প্রতিরোধ করার জন্য ডিজাইন করা হয়েছে, তা বর্ণনা করা।
- ব্যাখ্যা করা কেন consensus-এর ওপর নির্মিত ZooKeeper এবং etcd, একটি single Redis instance-এর তুলনায় distributed lock-এর জন্য শক্তিশালী correctness guarantee প্রদান করে।
- fencing token pattern বোঝা এবং কেন শুধুমাত্র একটি lock একটি distributed system-এ mutual exclusion নিশ্চিত করার জন্য যথেষ্ট নয়, তা বোঝা।
- একটি নির্দিষ্ট scenario-র correctness requirement অনুযায়ী উপযুক্ত distributed locking পদ্ধতি বেছে নেওয়া।

## স্ক্রিপ্ট (Script)

### শুরু / ভূমিকা (Hook / Intro)

একটি সাধারণ lock — আপনার পছন্দের programming language-এর একটি mutex — কাজ করে কারণ যেসব thread এটির জন্য প্রতিদ্বন্দ্বিতা করতে পারে, তারা সবাই একই process-এর ভেতরে থাকে এবং একই memory শেয়ার করে। যে মুহূর্তে আপনার একাধিক independent server থাকে, যাদের প্রত্যেকে আপনার application-এর নিজস্ব একটি copy চালাচ্ছে, তখন এই পুরো mechanism-টিই আর থাকে না। তবুও অন্তর্নিহিত সমস্যাটি দূর হয় না: আপনার প্রায়ই এখনও দরকার হয় যে অনেক process-এর মধ্যে ঠিক একটিই কোনো কাজ করুক — একটি scheduled email ঠিক একবার পাঠানো, একটি নির্দিষ্ট job ঠিক একবার process করা, একটি resource পরিবর্তন করার সময় তার ওপর exclusive access ধরে রাখা। এটাই হলো distributed locking, এবং আজ আমরা দুটি প্রধান পদ্ধতি নিয়ে আলোচনা করব — Redis-ভিত্তিক (Redlock) এবং consensus-ভিত্তিক (ZooKeeper/etcd) — সেই সাথে এমন একটি সূক্ষ্ম বিষয় যা প্রথমবার প্রায় সবাইকে ধরে ফেলে: শুধু কোনো lock "আছে" বললেই তা স্বয়ংক্রিয়ভাবে নিরাপদ হয়ে যায় না।

### কেন এটি কঠিন

একটি single process-এ, একটি mutex mutual exclusion নিশ্চিত করে কারণ operating system সরাসরি shared memory-র ওপর এটি প্রয়োগ করে — এই মুহূর্তে কে lock ধরে আছে তা নিয়ে কোনো অস্পষ্টতা থাকে না। একটি distributed system-এ, "lock"-কে অবশ্যই এমন কোনো বাহ্যিক জায়গায় থাকতে হবে যা প্রতিটি process দেখতে পারে — সাধারণত Redis-এর মতো একটি shared data store, অথবা ZooKeeper বা etcd-এর মতো একটি dedicated coordination service — এবং এর সাথে প্রতিটি interaction ঘটে একটি অনির্ভরযোগ্য network-এর মাধ্যমে, যার নিজস্ব latency, সম্ভাব্য partition, এবং সম্ভাব্য node failure রয়েছে। একটি distributed lock-কে যে মূল প্রশ্নের উত্তর দিতে হয় তা শুধু "কার কাছে lock আছে" নয় — বরং "আমরা এমন একজন lock holder-কে কীভাবে সামলাব যাকে আমরা আর reach করতে পারছি না, চিরতরে deadlock না হয়ে এবং দুইজন holder-কে একই সাথে exclusive access আছে বলে বিশ্বাস করতে না দিয়ে।"

### Redlock: একটি Redis-ভিত্তিক পদ্ধতি

সবচেয়ে সাধারণ lightweight পদ্ধতিটি Redis ব্যবহার করে: একটি lock acquire করা মানে হলো একটি key set করা (holder-কে চিহ্নিত করার জন্য একটি unique value সহ) যা একটি timeout-এর পরে স্বয়ংক্রিয়ভাবে expire হয়ে যায় — `SET lock_key unique_value NX PX 30000` এটিকে কেবল তখনই set করে যদি এটি ইতিমধ্যে বিদ্যমান না থাকে, এবং একটি ৩০-সেকেন্ডের expiry সহ। যদি process crash করে বা connectivity হারায়, তাহলে lock-টি চিরকাল ধরে থাকে না; এটি কেবল expire হয়ে যায় এবং অন্য একটি process এটি acquire করতে পারে। Lock release করলে key-টি delete হয়ে যায়, তবে কেবল তখনই যদি value এখনও মিলে যায় — এটি একটি process-কে ভুলবশত এমন একটি lock release করা থেকে বিরত রাখে যা সে আসলে আর ধরে নেই (যেমন, তার নিজের lock ইতিমধ্যে expire হয়ে যাওয়ার পরে এবং অন্য কেউ এটি acquire করার পরে)।

তবে একটি single Redis instance একটি single point of failure — যদি সেই Redis node ডাউন হয়ে যায়, তাহলে প্রতিটি lock অনুপলব্ধ হয়ে পড়ে, অথবা আরও খারাপ, mid-hold অবস্থায় হারিয়ে যেতে পারে। **Redlock**, যা Redis-এর নির্মাতা প্রস্তাব করেছিলেন, এই সমস্যার সমাধান করে একই lock acquisition-কে **পাঁচটি independent Redis instance**-এর বিরুদ্ধে চালিয়ে এবং lock acquired বিবেচনা করার আগে একটি tight time budget-এর মধ্যে একটি majority (পাঁচটির মধ্যে তিনটি)-কে সফল হতে হবে বলে শর্ত দিয়ে। ধারণাটি হলো, এমনকি যদি দুই-একটি Redis node ব্যর্থ হয় বা ধীর হয়, তবুও lock-টি নিরাপদে acquire এবং release করা যেতে পারে যতক্ষণ একটি majority একমত হয় — consensus algorithm থেকে "majority quorum" ধারণাটি ধার করে (Module 6 থেকে Raft মনে করুন) সম্পূর্ণ একটি consensus protocol ছাড়াই।

Redlock distributed systems মহলে সত্যিকার অর্থেই বিতর্কিত — Martin Kleppmann (*Designing Data-Intensive Applications*-এর লেখক)-এর একটি ব্যাপকভাবে উদ্ধৃত সমালোচনা যুক্তি দেয় যে Redlock-এর timing assumption গুলো (যে পাঁচটি node জুড়ে clock যুক্তিসঙ্গতভাবে synchronized থাকে, যে একটি process-এর নিজস্ব clock ঠিক ভুল মুহূর্তে দীর্ঘ pause অনুভব করে না, যেমন garbage collection থেকে) এমনভাবে লঙ্ঘিত হতে পারে যা দুইজন client-কে বিশ্বাস করাতে পারে যে তারা একই সাথে একই lock ধরে আছে। এর মানে এই নয় যে Redlock অকেজো — অনেক practical use case-এর জন্য (duplicate কাজ এড়ানো, adversarial condition-এর অধীনে নিখুঁত correctness নয়) এটি একটি যুক্তিসঙ্গত, দ্রুত, low-overhead টুল। তবে এটি জানা গুরুত্বপূর্ণ যে এটি একটি গাণিতিকভাবে সম্পূর্ণ নিশ্ছিদ্র mutual-exclusion guarantee নয়, যা গুরুত্বপূর্ণ হয়ে ওঠে যদি আপনার use case সত্যিকার অর্থেই দুইজন holder থাকার বিরল ক্ষেত্র সহ্য করতে না পারে।

### ZooKeeper / etcd: Consensus-ভিত্তিক Locking

synchronized clock-এর ওপর নির্ভর না করা correctness guarantee-র জন্য, ZooKeeper এবং etcd একটি মৌলিকভাবে ভিন্ন পদ্ধতি নেয়: এগুলো সরাসরি একটি consensus protocol-এর ওপর নির্মিত (ZooKeeper ZAB ব্যবহার করে, etcd Raft ব্যবহার করে — উভয়ই Module 6-এ ধারণাগতভাবে কভার করা হয়েছে), অর্থাৎ এই node-গুলোর একটি cluster state-এর একটি strongly-consistent, সম্মত view বজায় রাখে, এবং কে lock ধরে আছে তা নিয়ে কখনো দ্বিমত না হয়ে node-গুলোর একটি minority-র ব্যর্থতা সহ্য করে। একটি typical ZooKeeper-ভিত্তিক lock কাজ করে এভাবে যে প্রতিটি প্রতিদ্বন্দ্বী একটি shared path-এর নিচে একটি sequential, ephemeral node তৈরি করে; সবচেয়ে কম sequence number-যুক্ত প্রতিদ্বন্দ্বী lock ধরে রাখে, এবং বাকি সবাই তাদের ঠিক সামনের node-টি watch করে, এবং যখন তাদের পালা আসে তখন notify পায়। "Ephemeral" মানে হলো সেই client-এর session মারা গেলে (ZooKeeper cluster-এ heartbeat-এর মাধ্যমে detect করা হয়) node-টি স্বয়ংক্রিয়ভাবে সরিয়ে ফেলা হয় — তাই একজন crash হওয়া lock holder বাকি সবাইকে চিরকাল আটকে রাখে না, Redlock যেভাবে wall-clock timeout অনুমান করার ওপর নির্ভর করে সেভাবে না করে। এটি শক্তিশালী correctness কিনে নেয়, তার বিনিময়ে আপনার হয়তো ইতিমধ্যে থাকা একটি cache পুনর্ব্যবহার করার পরিবর্তে একটি dedicated, non-trivial coordination cluster চালানোর (এবং operate করার) খরচে।

### Fencing Token সমস্যা

এখানে সেই সূক্ষ্ম বিষয়টি রয়েছে যা এমনকি একটি "সঠিক" lock থাকা সত্ত্বেও মানুষকে ধরে ফেলে: একটি lock acquire করা এই guarantee দেয় না যে holder সময়মতো, নিরবচ্ছিন্নভাবে সেটির ওপর কাজ করবে। কল্পনা করুন একজন client একটি lock acquire করে, তারপর lock acquire করার *পরে* কিন্তু আসলে কোনো shared storage-এ লেখার জন্য এটি ব্যবহার করার *আগে* একটি দীর্ঘ pause অনুভব করে (garbage collection, একটি ধীর network, OS দ্বারা descheduled হওয়া)। ইতিমধ্যে, lock-টি হয়তো expire হয়ে গেছে এবং একজন দ্বিতীয় client দ্বারা acquire করা হয়ে গেছে, যে তার কাজ করে এবং সেটি release করে। প্রথম client-এর pause শেষ হলে, সে হয়তো resume করে এবং storage-এ লেখে — সম্পূর্ণভাবে অজ্ঞাত যে তার lock আর valid নেই, এমনকি দ্বিতীয় একজন client-এর অস্তিত্ব সম্পর্কেও তার কোনো ধারণা নেই। lock-এর অস্তিত্ব সঠিকভাবে যাচাই করা হয়েছিল, কিন্তু এর অনুমান (যে lock ধরে থাকা মানে কাজ করার জন্য exclusive সময়) মিথ্যা প্রমাণিত হলো। মানসম্মত সমাধান হলো একটি **fencing token**: প্রতিবার একটি lock grant করার সময়, এর সাথে একটি strictly increasing সংখ্যা আসে। client যখন shared storage-এ লেখে, তখন সে এই token-টি অন্তর্ভুক্ত করে, এবং storage system নিজেই এমন যেকোনো write প্রত্যাখ্যান করে যা ইতিমধ্যে দেখা কোনো token-এর চেয়ে পুরনো একটি token বহন করে — তাই এমনকি যদি একজন বিলম্বিত client তার lock কার্যকরভাবে expire হয়ে যাওয়ার পরে কাজ করার চেষ্টা করে, তার stale token প্রকৃত প্রভাবের মুহূর্তে প্রত্যাখ্যাত হয়, শুধু lock acquisition-এর মুহূর্তে নয়।

### বাস্তব-জগতের উদাহরণ (Real-World Example)

একটি distributed cron system-এর কথা ভাবুন যেখানে একটি scheduled job (ধরুন, "daily digest email পাঠানো") একাধিক worker instance-এর একটিতে চলে, এবং এটি কোনোভাবেই দুইবার চলতে পারবে না। আপনার team ইতিমধ্যে operate করা একটি Redis cluster-এর বিরুদ্ধে Redlock ব্যবহার করা দ্রুত set up করা যায় এবং যথেষ্ট ভালো যদি মাঝেমধ্যে একটি duplicate email একটি বিরক্তিকর ব্যাপার হয়, প্রকৃত ঘটনা না হয়। যদি এর পরিবর্তে এটি হতো "একটি subscription renewal-এর জন্য একজন customer-এর card charge করা" — যেখানে একটি duplicate execution একটি প্রকৃত আর্থিক এবং বিশ্বাসের সমস্যা — তাহলে আপনি ZooKeeper বা etcd-এর consensus-backed locking-এর শক্তিশালী guarantee চাইতেন, এবং আপনি সম্ভবত প্রকৃত charge-processing ধাপে একটি fencing token যোগ করতেন যাতে একটি expired lock-এর একটি বিলম্বিত, "zombie" holder-ও একটি প্রকৃত double-charge ঘটাতে না পারে।

### সংক্ষিপ্তসার (Recap)

Distributed locking-এর অস্তিত্ব রয়েছে কারণ সাধারণ in-process mutex independent machine জুড়ে coordinate করতে পারে না — lock-কে অবশ্যই একটি shared, network-accessible জায়গায় থাকতে হবে, এবং এমন একজন holder-কে সামলাতে হবে যে unreachable হয়ে যায়, deadlock বা double-granting না করে। Redlock একটি দ্রুত, practical lock-এর জন্য independent Redis instance-গুলোর একটি majority ব্যবহার করে, যার সাথে পরিচিত, বিতর্কিত timing-assumption দুর্বলতা রয়েছে। ZooKeeper এবং etcd dedicated coordination infrastructure চালানোর খরচে শক্তিশালী correctness guarantee দিতে consensus protocol ব্যবহার করে। এবং আপনি যে lock-ই ব্যবহার করুন না কেন, একটি fencing token — যা shared storage-এ প্রকৃতপক্ষে লেখার মুহূর্তে যাচাই করা একটি strictly increasing সংখ্যা — "lock ধরে ছিল" এবং "lock যা রক্ষা করছিল সেই action প্রকৃতপক্ষে নিরাপদে ঘটেছিল" এর মধ্যকার ফাঁকটি বন্ধ করে দেয়।

### এরপর কী (What's Next)

আমরা কভার করেছি কীভাবে distributed process-গুলো একটি resource-এর exclusive access coordinate করে। পরবর্তী video একটি সম্পর্কিত কিন্তু ভিন্ন সমস্যা নিয়ে আলোচনা করবে: কীভাবে একটি distributed system event-গুলো কোন *order*-এ ঘটেছিল তা নিয়ে একমত হয়, যখন এমন কোনো একক shared clock নেই যার ওপর প্রতিটি machine ভরসা করতে পারে।

## মূল বিষয়সমূহ (Key Takeaways)

- সাধারণ mutex machine জুড়ে coordinate করতে পারে না কারণ কোনো shared memory নেই — একটি distributed lock-কে অবশ্যই একটি বাহ্যিক, network-accessible store-এ থাকতে হবে এবং deadlock বা double-granting না করে unreachable holder সামলাতে হবে।
- Redlock একটি expiry সহ independent Redis instance-গুলোর একটি majority-র বিরুদ্ধে একটি lock acquire করে — দ্রুত এবং practical, কিন্তু এর পরিচিত, বিতর্কিত timing-assumption দুর্বলতা রয়েছে (clock skew, process pause)।
- ZooKeeper এবং etcd একটি consensus protocol (ZAB/Raft)-এর ওপর lock তৈরি করে, dedicated coordination infrastructure operate করার খরচে শক্তিশালী correctness guarantee প্রদান করে।
- একটি lock ধরে থাকা মানে নিরাপদ, নিরবচ্ছিন্ন action নিশ্চিত করা নয় — একটি process একটি lock acquire করার পরে pause হতে পারে এবং এটি কার্যকরভাবে expire হয়ে যাওয়ার পরে কাজ করতে পারে।
- Fencing token (shared storage-এ প্রকৃতপক্ষে লেখার মুহূর্তে যাচাই করা একটি strictly increasing সংখ্যা) এই ফাঁকটি বন্ধ করে দেয়, এমনকি একজন "zombie" lock holder থেকে আসা stale action-ও প্রত্যাখ্যান করে।
