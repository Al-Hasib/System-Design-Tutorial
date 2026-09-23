# Consistent Hashing Explained

Difficulty: Advanced

## Learning Objectives

- ব্যাখ্যা করা যে কেন naive `hash(key) mod N` পার্টিশনিং cluster resizing-এর সময় ভেঙে পড়ে।
- hash ring মডেল এবং কীভাবে key ও node গুলো এর উপর বসানো হয় তা বর্ণনা করা।
- virtual node কীভাবে load imbalance এবং heterogeneous capacity সমস্যা সমাধান করে তা ব্যাখ্যা করা।
- একটি node ring-এ join বা leave করলে কতগুলো key move করে তা পরিমাণগতভাবে যুক্তি দিয়ে বোঝা।
- consistent hashing-কে বাস্তব সিস্টেমের সাথে সংযুক্ত করা: DynamoDB, Cassandra, Riak, এবং CDN/load-balancer routing।

## Script

### Hook / Intro

একটা দৃশ্য কল্পনা করুন: আপনি একটি 10-node Memcached cluster চালাচ্ছেন, মাসের পর মাস ধরে এটি ভালোভাবেই চলছে, তারপর traffic বাড়ে, তাই আপনি আরও কয়েকটি node যোগ করেন। যেই মুহূর্তে আপনি এটি করেন, আপনার cache hit rate একদম তলানিতে নেমে যায়। প্রতিটি client হঠাৎ একই key-এর জন্য ভিন্ন ভিন্ন server-কে জিজ্ঞাসা করতে শুরু করে, আপনার origin database প্রচণ্ড চাপে পড়ে, এবং আপনি সারা বিকেল আপনার টিমকে বোঝাতে ব্যয় করেন যে কেন "শুধু capacity যোগ করা" আউটেজ প্রতিরোধ করার বদলে সেটির কারণ হয়ে দাঁড়াল।

এটাই naive-hashing failure mode, এবং ঠিক এই সমস্যাটি সমাধানের জন্যই consistent hashing আবিষ্কৃত হয়েছিল। আপনি ইতিমধ্যে sharding এবং partitioning জানেন, আপনি ইতিমধ্যে CAP theorem-এর trade-off গুলোও জানেন — আজ আমরা এক ধাপ গভীরে গিয়ে সেই আসল mechanism নিয়ে কথা বলব যা DynamoDB, Cassandra, এবং প্রায় সব বড় CDN-কে সম্পূর্ণ data reshuffle ছাড়াই node যোগ ও বাদ দিতে দেয়। চলুন শুরু করি।

### The Problem with Naive Hashing (mod N)

N-টি node জুড়ে key বিতরণ করার সবচেয়ে সহজ উপায় হলো `node = hash(key) mod N`। এটি সহজ, একটি ভালো hash function-এর জন্য এটি uniform distribution দেয়, এবং fixed cluster size-এর জন্য এটি ঠিকঠাক কাজ করে। সমস্যা হয় যখন N পরিবর্তিত হয়।

যদি আপনি N থেকে N+1 node-এ যান, তাহলে modulus পরিবর্তিত হয়, এবং সেটি প্রায় প্রতিটি key-এর assignment বদলে দেয়। যে key `hash(key) mod 10 = 7`-এ ছিল সেটি এখন `hash(key) mod 11 = 3`-এ চলে যেতে পারে। mod-N এবং mod-(N+1) assignment-এর মধ্যে কোনো structural সম্পর্ক নেই — এটি কার্যত একটি সম্পূর্ণ নতুন random remapping। সবচেয়ে খারাপ ক্ষেত্রে, প্রায় সব key-এর `(N-1)/N` অংশ move করে। একটি 10-node cluster-এর জন্য, এর মানে হলো একটি node পরিবর্তনের জন্য প্রায় 90% data reshuffle হওয়া। একটি cache-এর জন্য, এর মানে হলো প্রায় সম্পূর্ণ cold start। একটি sharded database-এর জন্য, এর মানে হলো শুধু capacity যোগ করার জন্য প্রায় সম্পূর্ণ dataset network জুড়ে move করা — যা incrementally scale করার আসল উদ্দেশ্যকেই ব্যর্থ করে দেয়।

এটাই মূল সমস্যা যা consistent hashing সমাধান করে: আমরা চাই node যোগ বা অপসারণ যেন শুধুমাত্র সেই node-এর মালিকানাধীন key-গুলোকেই প্রভাবিত করে, আর কিছুকে নয়।

### The Hash Ring

কৌশলটি হলো hashing-কে "একটি bucket index বেছে নেওয়া" হিসেবে চিন্তা করা বন্ধ করে, এটিকে "space-এ একটি point বেছে নেওয়া" হিসেবে চিন্তা করা শুরু করা। Consistent hashing node এবং key উভয়কেই একই hash space-এ ম্যাপ করে — সাধারণত 0 থেকে 2^32 বিয়োগ 1 পর্যন্ত একটি fixed range — এবং সেই space-কে একটি ring হিসেবে বিবেচনা করে: maximum value-এর পরে, আপনি আবার শূন্যে ফিরে আসেন। একটি clock face কল্পনা করুন, শুধু 12 ঘণ্টার বদলে আপনার কাছে বিলিয়ন বিলিয়ন position রয়েছে।

প্রতিটি node hash করা হয় — প্রায়ই node-এর IP এবং port ব্যবহার করে — এবং ফলস্বরূপ position-এ ring-এর উপর বসানো হয়। প্রতিটি key একইভাবে hash করা হয় এবং ring-এর উপর বসানো হয়। কোন node কোন key-এর মালিক তা বের করার জন্য, আপনি key-এর position থেকে clockwise দিকে হাঁটতে থাকেন যতক্ষণ না প্রথম node-এ পৌঁছান। সেই node-ই key-এর মালিক। প্রতিটি node কার্যত ring-এর সেই arc-এর মালিক যা তার এবং counter-clockwise দিকে আগের node-এর মধ্যে থাকে।

এখন এখানেই বোঝা যাক কেন এটি rebalancing সমস্যার সমাধান করে। যখন আপনি একটি নতুন node যোগ করেন, এটি ring-এর কোনো position-এ পড়ে, এবং এটি শুধুমাত্র তার ঠিক counter-clockwise দিকের arc-এর key-গুলো "চুরি" করে — যেসব key আগে clockwise দিকের পরবর্তী node-এর ছিল। বাকি সব node-এর মালিকানা সম্পূর্ণ অক্ষত থাকে। একইভাবে, যখন একটি node সরানো হয়, শুধু তার key-গুলোই move করার প্রয়োজন হয়, এবং সেগুলো সবই clockwise দিকের পরবর্তী node-এ চলে যায় — বাকি সবাই তাদের যা ছিল তাই ধরে রাখে। প্রায় 100% key remap করার বদলে, আপনি প্রায় K/N key remap করছেন, যেখানে K হলো মোট key-সংখ্যা এবং N হলো node-সংখ্যা। একটি 10-node ring-এ একটি node যোগ করলে প্রায় 1/11 অংশ data move করে, 90% নয়। এটাই এক বাক্যে পুরো value proposition: ন্যূনতম বিঘ্ন, যা পরিবর্তনের আকারের সমানুপাতিক, cluster-এর আকারের নয়।

এটাও স্পষ্ট করে বলা দরকার যে এখানে "consistent" মানে কী — এটি CAP অর্থে strong consistency নয়, এটি routing-এর consistency: independent client-রা, একে অপরের সাথে কথা না বলেই, একই key-এর জন্য একই node গণনা করবে, যতক্ষণ তাদের কাছে ring membership-এর একই view থাকে। এই property-ই ring-কে একটি central lookup service-এর প্রয়োজন ছাড়াই একটি decentralized routing mechanism হিসেবে ব্যবহারযোগ্য করে তোলে।

### Virtual Nodes and Load Balancing

naive ring-এর একটি সমস্যা আছে: যদি আপনি শুধুমাত্র প্রতিটি physical node একবার বসান, তাহলে arc-গুলো random আকারের হয়। অল্প সংখ্যক node নিয়ে, একটি বড় hash space-এ random placement অত্যন্ত অসম arc length তৈরি করতে পারে — একটি node হয়তো keyspace-এর 40% মালিক হতে পারে আবার অন্যটি মাত্র 5%। আপনি সহজেই একটি বেশি ক্ষমতাসম্পন্ন machine-কে একটি ছোট machine-এর চেয়ে বেশি load দিতেও পারবেন না।

এর সমাধান হলো virtual node, যাকে কখনো কখনো ring-এ "vnodes" বা replica বলা হয়। একটি physical node-কে একবার hash করার বদলে, আপনি এটিকে বিভিন্ন suffix দিয়ে একাধিকবার hash করেন — node-A#1, node-A#2, node-A#3, ইত্যাদি — এবং সেই সব point ring-এর উপর বসিয়ে দেন। প্রতিটি physical node এখন একটি বড় continuous arc-এর বদলে অনেকগুলো ছোট, ছড়ানো arc-এর মালিক হয়। প্রতিটি physical node-এ যথেষ্ট virtual node থাকলে — production system-এ সাধারণত 100 থেকে 200-এর মধ্যে ব্যবহৃত হয়, যদিও কিছু ক্ষেত্রে আরও বেশি হয় — law of large numbers কাজ করা শুরু করে এবং hash function যেখানেই জিনিসগুলো বসাক না কেন, প্রতিটি physical node-এর মালিকানাধীন মোট keyspace পরিসংখ্যানগতভাবে একটি সমান share-এর দিকে converge করে।

Virtual node heterogeneous-hardware সমস্যাও সরাসরি সমাধান করে: যদি একটি machine-এর RAM বা disk অন্যটির দ্বিগুণ হয়, আপনি শুধু তাকে দ্বিগুণ virtual node দিতে পারেন। ফলে এটি প্রায় দ্বিগুণ arc length-এর মালিক হয় এবং সেই অনুযায়ী প্রায় দ্বিগুণ data ও traffic বহন করে — routing logic-এ কোনো বিশেষ case ছাড়াই।

আর virtual node আসলে failure-recovery-এর গল্পকেও উন্নত করে, যা নিয়ে আমরা পরবর্তীতে আলোচনা করব: যখন একটি node ব্যর্থ হয়, তার সব load ঠিক একটি প্রতিবেশীর উপর পড়ার বদলে, এটি অনেকগুলো ভিন্ন physical node জুড়ে ছড়িয়ে পড়ে, কারণ ব্যর্থ node-এর প্রতিটি virtual point-এর ভিন্ন clockwise প্রতিবেশী থাকে।

### Handling Node Failure and Growth

চলুন দুটো operation নিয়ে concrete-ভাবে আলোচনা করি।

Node addition: নতুন node তার virtual node position-এর set গণনা করে এবং সেগুলো ring-এ insert করে। প্রতিটি virtual point-এর জন্য, এটি ঠিক তার আগের arc-এর মালিকানা নেয়, যা আগে যেই node clockwise-এ পরবর্তী ছিল তার ছিল। সেই নির্দিষ্ট key range-এর data পুরনো owner থেকে নতুন node-এ stream হয়। cluster-এর অন্য কোনো node-কে কিছুই করতে হয় না। client বা coordinator তাদের ring membership-এর view refresh করার সাথে সাথেই read এবং write routing আপডেট হয়ে যায়।

Node removal, তা পরিকল্পিত decommissioning হোক বা প্রকৃত failure, এটি এর প্রতিফলন: সেই node-এর virtual point ring থেকে সরানো হয়, এবং তার arc-গুলো প্রতিটি point-এর জন্য বর্তমানে যে node clockwise-এ পরবর্তী প্রতিবেশী তার সাথে মিশে যায়। যেহেতু virtual node একটি physical node-এর মালিকানা অনেক প্রতিবেশীর মধ্যে ছড়িয়ে দেয়, সেই failover load একটি দুর্ভাগা প্রতিবেশীর উপর কেন্দ্রীভূত না হয়ে পুরো cluster জুড়ে বিতরণ হয়।

বাস্তব সিস্টেমে এর সাথে availability-র জন্য replication যুক্ত করা থাকে: DynamoDB এবং Cassandra-স্টাইলের সিস্টেমগুলো ring-এর immediate successor-এ একটি key-এর একটিমাত্র copy রাখে না — তারা clockwise দিকে হেঁটে পরবর্তী R-টি ভিন্ন physical node-এ replicate করে, ফলে একটি node-এর failure মানে data loss নয়, এর মানে হলো ring সেরে ওঠার সময় replica-তে fall back করা। এখানেই আপনার ইতিমধ্যে জানা CAP trade-off গুলোও আবার সামনে আসে: একটি write acknowledge করতে কতগুলো replica-র প্রয়োজন, partition চলাকালীন replica-গুলোর মধ্যে conflict কীভাবে resolve হয় — vector clock, last-write-wins, read-repair — সবই এই ring-based placement layer-এর উপরে বসে থাকে।

আরেকটি সূক্ষ্ম বিষয়: ring membership নিজেই node এবং client জুড়ে সম্মত হতে হবে, অথবা অন্তত শেষ পর্যন্ত converge করতে হবে। Cassandra-র মতো সিস্টেম gossip protocol ব্যবহার করে যাতে প্রতিটি node শেষমেশ বর্তমান ring topology জানতে পারে; DynamoDB-স্টাইলের সিস্টেমগুলোও একইভাবে membership পরিবর্তন প্রচার করে যাতে routing পুরো cluster জুড়ে consistent থাকে, এমনকি topology পরিবর্তনের সময় সংক্ষিপ্ত সময়ের জন্য মতানৈক্য থাকলেও।

### Real-World Example

আপনি বেশিরভাগ বড়-মাপের distributed data store এবং routing layer-এর নেপথ্যে consistent hashing খুঁজে পাবেন। Amazon-এর Dynamo paper, যা আমরা resources-এ link করব, এই কৌশলটিকে database-এর জন্য জনপ্রিয় করে তোলা সিস্টেম — এটি partitioning-এর জন্য virtual node সহ একটি hash ring ব্যবহার করে, এবং DynamoDB সেই ধারা থেকেই এসেছে। Apache Cassandra একই ধারণা স্পষ্টভাবে ব্যবহার করে: এর partitioner row key-গুলোকে একটি token ring-এ ম্যাপ করে, এবং virtual node (vnodes) load সমানভাবে ছড়ানো এবং bootstrap ও repair দ্রুত করার জন্য একটি standard configuration। Riak, আরেকটি Dynamo-inspired store, এর partitioning scheme-এর জন্য প্রায় একই রকম consistent-hashing ring vnode-সহ ব্যবহার করে।

শুধু database-ই নয়। CDN এবং load balancer-ও consistent hashing ব্যবহার করে request-কে origin server বা cache node-এ route করার জন্য, যাতে একই request key — একটি URL বা client identifier — সব সময় একই backend-এ পৌঁছায়, যা cache locality সর্বোচ্চ করে, এবং যাতে একটি edge বা backend node যোগ বা অপসারণ একবারে পুরো cache-এর routing decision অকার্যকর না করে দেয়। কিছু client-side load balancing library এবং service mesh ঠিক এই কারণেই consistent hashing ব্যবহার করে, যখন তাদের sticky routing দরকার যা পরিবর্তনশীল backend pool-কেও সহ্য করতে পারে।

### Recap

চলুন সবকিছু একসাথে বাঁধি। Naive mod-N hashing cluster-এর আকার পরিবর্তনের সময় প্রায় সবকিছু remap করে, কারণ modulus নিজেই পরিবর্তিত হয়। Consistent hashing এটি সমাধান করে node এবং key-কে একটি shared ring-এ ম্যাপ করে এবং প্রতিটি key-কে clockwise দিকের পরবর্তী node-এ assign করে, ফলে topology পরিবর্তনের কাছাকাছি arc-ই শুধু প্রভাবিত হয় — প্রায় সব key-এর বদলে মাত্র K/N key move করে। Virtual node ফলস্বরূপ load-imbalance সমস্যা সমাধান করে প্রতিটি physical node-কে একটি বড় arc-এর বদলে অনেকগুলো ছোট, ছড়ানো arc দিয়ে, যা আপনাকে capacity weight করতে এবং failover load পুরো cluster জুড়ে ছড়িয়ে দিতেও দেয়। আর এটি একটি routing-consistency property, CAP-consistency property নয় — এটাই independent client-দের কোনো central coordinator ছাড়াই একটি key কোথায় থাকে তা নিয়ে একমত হতে দেয়। এটাই DynamoDB, Cassandra, Riak, এবং CDN ও load-balancer routing-এর বিশাল একটি অংশের নেপথ্যের mechanism।

### What's Next

এখন যেহেতু আপনি জানেন distributed system কীভাবে সিদ্ধান্ত নেয় একটি request বা key *কোথায়* যাবে, স্বাভাবিক পরবর্তী প্রশ্ন হলো তারা কীভাবে সিদ্ধান্ত নেয় load-এর মধ্যে একটি request-কে *আদৌ* ঢুকতে দেওয়া হবে কিনা। পরবর্তী video-তে, আমরা rate limiting algorithm নিয়ে আলোচনা করব — token bucket, leaky bucket, fixed এবং sliding window counter — এবং কীভাবে সিস্টেমগুলো এগুলোকে আপনার এইমাত্র শেখা distributed routing-এর সাথে combine করে পুরো node-এর fleet জুড়ে consistent-ভাবে limit প্রয়োগ করে। সেখানে দেখা হবে।

## Key Takeaways

- Naive `hash(key) mod N` যেকোনো cluster resize-এ সব key-এর `(N-1)/N` পর্যন্ত remap করে, কারণ modulus নিজেই পরিবর্তিত হয়।
- Consistent hashing node এবং key-কে একটি shared hash ring-এ ম্যাপ করে; একটি key তার position থেকে clockwise দিকে হেঁটে পাওয়া প্রথম node-এর অন্তর্গত।
- একটি node যোগ বা অপসারণ শুধুমাত্র সংলগ্ন arc-কে প্রভাবিত করে, প্রায় সব key-এর বদলে প্রায় K/N key move করে।
- Virtual node প্রতিটি physical node-কে অনেকগুলো ছড়ানো arc দেয়, যা সমান load distribution তৈরি করে এবং আপনাকে capacity অনুযায়ী node-কে weight করতে দেয়।
- এখানে "Consistent" মানে routing consistency (independent client-রা key ownership নিয়ে একমত), CAP-স্টাইলের data consistency নয়।
- Production system সাধারণত প্রতি physical node-এ প্রায় 100-200 virtual node ব্যবহার করে, availability-র জন্য এর সাথে replication (clockwise-এ N replica) যুক্ত থাকে।
- DynamoDB, Cassandra, এবং Riak সবাই virtual node-সহ ring-based consistent hashing ব্যবহার করে; CDN এবং load balancer এটি sticky, resize-tolerant routing-এর জন্য ব্যবহার করে।
