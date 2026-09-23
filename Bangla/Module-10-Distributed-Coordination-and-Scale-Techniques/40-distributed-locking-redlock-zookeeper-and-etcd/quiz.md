# Practice ও Interview প্রশ্ন

**১. একটি সাধারণ in-process mutex কেন একাধিক server জুড়ে access coordinate করতে পারে না?**
একটি mutex operating system-এর ওপর নির্ভর করে যা একটি single process-এর মধ্যে shared memory-র ওপর exclusion প্রয়োগ করে। একাধিক server memory শেয়ার করে না, তাই একটি mutex-এর প্রকৃতপক্ষে synchronize করার মতো কিছুই নেই — lock state-কে অবশ্যই প্রতিটি process-এর কাছে বাহ্যিকভাবে দৃশ্যমান কোনো জায়গায় থাকতে হবে, যা একটি অনির্ভরযোগ্য network-এর মাধ্যমে access করা হয়।

**২. Redlock কীভাবে সিদ্ধান্ত নেয় যে একটি lock সফলভাবে acquire হয়েছে, তা ব্যাখ্যা করুন।**
এটি একাধিক independent Redis instance-এর (সাধারণত ৫টি) বিরুদ্ধে একই lock (একটি unique value এবং expiry-সহ একটি key) acquire করার চেষ্টা করে এবং lock-টি তখনই granted বলে বিবেচনা করে যখন একটি bounded time window-এর মধ্যে একটি majority (যেমন, ৫টির মধ্যে ৩টি) সফল হয় — consensus algorithm থেকে majority-quorum ধারণাটি ধার করে।

**৩. Martin Kleppmann Redlock-এর বিরুদ্ধে যে মূল সমালোচনা করেছেন তা কী?**
যে Redlock-এর safety timing assumption-এর ওপর নির্ভর করে — Redis instance-গুলো জুড়ে bounded clock drift এবং ভুল মুহূর্তে দীর্ঘ process pause না থাকা (যেমন garbage collection) — যা বাস্তব system-এ guaranteed নয়, অর্থাৎ বিরল ক্ষেত্রে দুইজন client উভয়েই বিশ্বাস করতে পারে যে তারা একই lock ধরে আছে।

**৪. একটি single Redis instance বা এমনকি Redlock-এর চেয়েও কেন ZooKeeper এবং etcd locking-এর জন্য শক্তিশালী correctness guarantee প্রদান করে?**
এরা সরাসরি একটি consensus protocol-এর ওপর নির্মিত (ZooKeeper-এর জন্য ZAB, etcd-এর জন্য Raft), যা independent instance এবং timing assumption-এর ওপর নির্ভর করার বদলে একটি cluster জুড়ে একটি strongly-consistent, সম্মত state view বজায় রাখে এবং কে lock ধরে আছে তা নিয়ে কখনো দ্বিমত না হয়ে node-গুলোর একটি minority সহ্য করে।

**৫. একটি timeout অনুমানের ওপর নির্ভর না করে একটি ZooKeeper-ভিত্তিক lock কীভাবে detect করে যে একজন lock holder crash করেছে?**
Lock holder-এর node "ephemeral" হিসেবে তৈরি করা হয়, যা তার client session-এর সাথে যুক্ত। যদি client-এর session মারা যায় (ZooKeeper cluster-এ missed heartbeat-এর মাধ্যমে detect করা হয়), ephemeral node-টি স্বয়ংক্রিয়ভাবে সরিয়ে ফেলা হয়, যা সাথে সাথে সারিতে পরবর্তী প্রতিদ্বন্দ্বীকে জানিয়ে দেয় যে lock-টি উপলব্ধ — একটি উপযুক্ত timeout সময়কাল অনুমান করার প্রয়োজন ছাড়াই।

**৬. এমন একটি scenario বর্ণনা করুন যেখানে "একটি lock ধরে রাখা" এই guarantee দেয় না যে সুরক্ষিত action নিরাপদে ঘটবে।**
একজন client একটি lock acquire করে, তারপর এটি acquire করার পরে কিন্তু এর ওপর কাজ করার আগে একটি দীর্ঘ pause অনুভব করে (GC, network delay, descheduled হওয়া)। ইতিমধ্যে, lock-টি expire হয়ে যায় এবং একজন দ্বিতীয় client এটি acquire করে, তার কাজ করে, এবং release করে। প্রথম client resume করলে, সে হয়তো এমনভাবে কাজ করে যেন সে এখনও নিরাপদে lock ধরে আছে, না জেনে যে একজন দ্বিতীয় client ইতিমধ্যে কাজ করে ফেলেছে — lock check সফল হয়েছিল, কিন্তু এই অনুমান যে এটি ধরে রাখা মানে কাজ করার জন্য exclusive সময়, তা লঙ্ঘিত হয়েছিল।

**৭. একটি fencing token কী, এবং এটি প্রশ্ন ৬-এর সমস্যাটি কীভাবে সমাধান করে?**
একটি fencing token হলো প্রতিটি lock grant-এর সাথে issue করা একটি strictly increasing সংখ্যা। সুরক্ষিত resource (যেমন, একটি storage system) ইতিমধ্যে গ্রহণ করা সর্বোচ্চ token ট্র্যাক করে এবং তার চেয়ে পুরনো token বহনকারী যেকোনো action প্রত্যাখ্যান করে — তাই এমনকি একজন বিলম্বিত client যে তার lock কার্যকরভাবে expire হয়ে যাওয়ার পরে resume করে, তার stale action প্রকৃত প্রভাবের মুহূর্তে প্রত্যাখ্যাত হয়, শুধু lock-acquisition-এর মুহূর্তে নয়।

**৮. Scenario: আপনার একাধিক worker instance-এ চলা একটি কম-ঝুঁকির scheduled job-এর (যেমন, একটি cache refresh করা) duplicate execution প্রতিরোধ করতে হবে। আপনি কোন locking পদ্ধতি বেছে নেবেন, এবং কেন?**
Redis-এর বিরুদ্ধে Redlock — এটি দ্রুত, low-operational-overhead, এবং এখানে "যথেষ্ট ভালো": একটি কম-ঝুঁকির job-এর মাঝেমধ্যে বিরল duplicate execution এই ক্ষেত্রে একটি dedicated ZooKeeper/etcd cluster দাঁড় করানোর বদলে আপনি সম্ভবত ইতিমধ্যে চালান এমন infrastructure পুনর্ব্যবহারের সরলতার জন্য একটি গ্রহণযোগ্য বিনিময়।

**৯. Scenario: আপনাকে নিশ্চিত করতে হবে যে একটি financial transaction (যেমন, একটি subscription charge process করা) ঠিক একবার execute হয়, এমনকি process pause বা network সমস্যার অধীনেও। শুধু একটি lock acquire করার বাইরে আপনি কী যোগ করবেন?**
Redlock একা-র চেয়ে শক্তিশালী correctness-এর জন্য একটি consensus-backed lock (ZooKeeper বা etcd), এবং customer-এর card charge করার প্রকৃত মুহূর্তে যাচাই করা একটি fencing token — যাতে একটি expired lock-এর একজন বিলম্বিত/"zombie" holder-ও একটি প্রকৃত duplicate charge ঘটাতে না পারে, যেহেতু payment system নিজেই একটি stale token প্রত্যাখ্যান করবে।

**১০. সত্য নাকি মিথ্যা: একবার একটি distributed lock সঠিকভাবে acquire হলে, এটি যে action সুরক্ষিত করে তা কোনো আরও সতর্কতা ছাড়াই নিরাপদে execute হওয়ার guarantee থাকে।**
মিথ্যা। Lock acquire করা শুধু প্রমাণ করে যে client acquisition-এর মুহূর্তে exclusive access ধরে রেখেছিল — action প্রকৃতপক্ষে চলার আগে একটি পরবর্তী pause সেই exclusivity-কে ফুরিয়ে যেতে দিতে পারে। Safety নিশ্চিত করতে action-এর মুহূর্তে fencing token (বা সমতুল্য mechanism) প্রয়োজন, শুধু lock-acquisition-এর মুহূর্তে নয়।
