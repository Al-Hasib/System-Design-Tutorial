# অনুশীলন ও ইন্টারভিউ প্রশ্ন

**১. CAP theorem-এর তিনটি অক্ষর কীসের প্রতিনিধিত্ব করে, এবং প্রতিটি আসলে কী guarantee দেয়?**
- Consistency: প্রতিটি read সবচেয়ে সাম্প্রতিক write ফেরত দেয় (অথবা একটি error), যেন ডেটার একটিমাত্র কপি আছে।
- Availability: একটি non-failed node-এ প্রতিটি request একটি response পায়, যদিও এতে অবশ্যই সাম্প্রতিকতম ডেটা থাকবে না।
- Partition Tolerance: node-গুলোর মধ্যে network message drop বা delay হলেও সিস্টেম কাজ চালিয়ে যায়।

**২. CAP theorem কি বোঝায় যে একটি সিস্টেম সবসময় "CP" বা "AP"?**
না। CAP theorem শুধু বর্ণনা করে একটি সিস্টেম *একটি সক্রিয় network partition-এর সময়* কী করে। স্বাভাবিক কার্যক্রমের সময়, একটি সুস্থ network দিয়ে, একটি ভালোভাবে ডিজাইন করা সিস্টেম consistency এবং availability দুটোই দিতে পারে। একটি সিস্টেমকে "CP" বা "AP" বলা হলো একটি সংক্ষিপ্ত রূপ, যা বর্ণনা করে যে একটি partition ঘটলে এটি বিশেষভাবে কী ত্যাগ করার সিদ্ধান্ত নেয়।

**৩. কেন "CA" (Partition Tolerance ছাড়া Consistency + Availability) distributed system-এর জন্য একটি বাস্তবসম্মত বিকল্প নয়?**
কারণ একটি প্রকৃত network-এর মাধ্যমে যোগাযোগকারী একের বেশি node সহ যেকোনো সিস্টেম শেষ পর্যন্ত একটি partition অভিজ্ঞতা লাভ করবে — একটি হারানো packet, একটি কাটা ক্যাবল, একটি ধীর/unreachable process। partition ঘটা থেকে আপনি বেরিয়ে আসতে পারেন না, তাই partition tolerance একটি প্রদত্ত বিষয়; আসল সিদ্ধান্তটি হলো একটি partition ঘটলে কী (C বা A) ত্যাগ করবেন। CA কার্যকরভাবে শুধুমাত্র single-node, non-distributed সিস্টেমের জন্য বিদ্যমান।

**৪. CP-ঝোঁকা দুটি সিস্টেমের উদাহরণ দিন এবং তাদের design পছন্দ ব্যাখ্যা করুন।**
ZooKeeper এবং etcd (এবং HBase) হলো CP: একটি partition-এর সময় তারা এমন request সার্ভ করতে অস্বীকার করে যা তারা নিশ্চিত করতে পারে না যে বাকি cluster-এর সাথে সামঞ্জস্যপূর্ণ। এটি তাদের ব্যবহারক্ষেত্রের সাথে মানানসই — leader election, configuration, এবং coordination — যেখানে দুটি node সত্য নিয়ে দ্বিমত পোষণ করলে সিস্টেমের অন্য অংশে গুরুতর bug তৈরি হবে।

**৫. AP-ঝোঁকা দুটি সিস্টেমের উদাহরণ দিন এবং তাদের design পছন্দ ব্যাখ্যা করুন।**
Cassandra এবং DynamoDB হলো AP: partition-এর দুই পাশেই তারা read এবং write সার্ভ করা চালিয়ে যায়, মেনে নিয়ে যে ডেটা সাময়িকভাবে ভিন্ন হয়ে যেতে পারে, এবং পরে conflict মিলিয়ে নেয় (যেমন, vector clocks বা last-write-wins-এর মাধ্যমে)। এটি উচ্চ-ট্রাফিক, availability-সংবেদনশীল ব্যবহারক্ষেত্রের সাথে মানানসই যেমন shopping cart বা social feed, যেখানে একটি error page একটি সংক্ষিপ্ত বাসি ডেটার চেয়ে খারাপ।

**৬. CAP theorem-এর কোন ফাঁক PACELC সমাধান করে?**
CAP কোনো partition না থাকলে সিস্টেম আচরণ সম্পর্কে কিছুই বলে না — যা বেশিরভাগ সিস্টেমের জন্য প্রায় পুরো সময়ই। PACELC যোগ করে: "Else (কোনো partition নেই), Latency এবং Consistency-র মধ্যে বেছে নিন," যা একটি write নিশ্চিত করার আগে বা একটি read সার্ভ করার আগে replica acknowledgment-এর জন্য কতক্ষণ অপেক্ষা করবেন সেই প্রাত্যহিক trade-off ধারণ করে।

**৭. PACELC স্পেকট্রামে DynamoDB, Cassandra, এবং MongoDB (majority read/write concern)-কে শ্রেণীবদ্ধ করুন।**
- DynamoDB: PA/EL — একটি partition-এর সময় availability-কে অগ্রাধিকার দেয়, এবং স্বাভাবিক কার্যক্রমের সময় নিম্ন latency (eventual consistency)-কে অগ্রাধিকার দেয়, একটি opt-in strongly consistent read mode সহ।
- Cassandra: default-এ PA/EL, কিন্তু consistency level (ONE, QUORUM, ALL)-এর মাধ্যমে প্রতি query-তে টিউনযোগ্য।
- MongoDB (majority concern): PC/EC — একটি partition-এর সময় এবং স্বাভাবিক কার্যক্রমের সময়, উভয় ক্ষেত্রে consistency-কে অগ্রাধিকার দেয়, latency/availability-র খরচে।

**৮. আপনি একটি বৈশ্বিক shopping cart সার্ভিস ডিজাইন করছেন। আপনি কি CP নাকি AP-এর দিকে ঝুঁকবেন, এবং কেন?**
AP। একজন গ্রাহকের একটি error দেখার বা তাদের cart-এ item যোগ করতে না পারার খরচ (হারানো বিক্রি, খারাপ অভিজ্ঞতা) সাধারণত পরে মাঝেমধ্যে একটি ছোটখাটো cart conflict মিলিয়ে নেওয়ার খরচের (যেমন, একটি stock-শেষ হয়ে যাওয়া item সংক্ষিপ্তভাবে available দেখানো) চেয়ে খারাপ। এটি DynamoDB তৈরি করার জন্য Amazon-এর নিজস্ব মূল যুক্তির সাথে মিলে যায়।

**৯. আপনি একটি banking system-এর জন্য একটি ledger ডিজাইন করছেন যা account-গুলোর মধ্যে টাকা স্থানান্তর করে। আপনি কি CP নাকি AP-এর দিকে ঝুঁকবেন, এবং কেন?**
CP। একটি অসামঞ্জস্যপূর্ণ balance read করার অনুমতি দেওয়া, বা replica জুড়ে funds নিশ্চিত না করে একটি transfer চালিয়ে যেতে দেওয়া, double-spending বা ভুল balance-এর ঝুঁকি তৈরি করে, যা একটি গুরুতর আর্থিক এবং নিয়ন্ত্রক সমস্যা। একটি partition-এর সময় সংক্ষেপে একটি transaction প্রত্যাখ্যান বা বিলম্বিত করা, ভুল, ফিরিয়ে আনা কঠিন এমন একটি আর্থিক অবস্থার ঝুঁকি নেওয়ার চেয়ে ভালো।

**১০. একজন সহকর্মী বলেন, "Cassandra একটি AP database, তাই এটি কখনো strongly consistent হতে পারে না।" এটা কি সঠিক?**
পুরোপুরি নয়। Cassandra টিউনযোগ্য: যদিও এর default availability এবং কম latency-কে অগ্রাধিকার দেয়, আপনি প্রতি-query ভিত্তিতে consistency level QUORUM বা ALL-এ সেট করতে পারেন অনেক বেশি শক্তিশালী consistency guarantee পেতে, কিছু availability/latency-র বিনিময়ে। "AP" Cassandra-র সাধারণ অবস্থান এবং design center বর্ণনা করে, এটি কী করতে পারে তার একটি কঠোর সীমা নয়।

**১১. Network split-এর সময় একটি CP এবং একটি AP system-এর মধ্যে পার্থক্যের জন্য একটি বাস্তব-জীবনের, non-technical analogy কী?**
একটি CP system একটি ব্যাংক ভল্টের মতো যা খুলতে দুইজন manager-এর উপস্থিতি প্রয়োজন — যদি একজনের সাথে যোগাযোগ করা না যায়, ভল্টটি যতই অসুবিধাজনক হোক না কেন, তালাবদ্ধই থাকে (সঠিকতাকে অগ্রাধিকার দিয়ে)। একটি AP system একটি দোকানের চেইনের মতো যা ইন্টারনেট আউটেজের সময় একটি স্থানীয় কাগজের খাতায় gift card বিক্রি চালিয়ে যায় এবং সংযোগ ফিরে এলে headquarters-এর সাথে balance মিলিয়ে নেয় (ব্যবসার জন্য খোলা থাকাকে অগ্রাধিকার দিয়ে)।

**১২. ইন্টারভিউয়াররা কেন CAP/PACELC নিয়ে চিন্তিত থাকেন, যদিও বেশিরভাগ ইঞ্জিনিয়ার কখনো স্পষ্টভাবে একটা "CAP mode" configure করেন না?**
কারণ এই trade-off গুলো প্রায় প্রতিটি distributed systems design সিদ্ধান্তে implicitly প্রকাশ পায় — একটি database বেছে নেওয়া, একটি write concern/consistency level সেট করা, একটি downstream service timeout কীভাবে সামলাবেন সেই সিদ্ধান্ত, বা retry/replication logic ডিজাইন করা। CAP/PACELC বোঝা আপনাকে সেই সিদ্ধান্তগুলো নিয়ে স্পষ্টভাবে চিন্তা করার এবং সেগুলোকে যুক্তিসঙ্গত করার শব্দভাণ্ডার এবং মানসিক মডেল দেয়, দুর্ঘটনাক্রমে সেগুলো নেওয়ার বদলে।
