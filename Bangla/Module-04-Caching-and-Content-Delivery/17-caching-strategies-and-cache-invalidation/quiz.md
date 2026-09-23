# অনুশীলন ও ইন্টারভিউ প্রশ্ন

**১. cache hit ratio কী, এবং কেন এমনকি একটি উচ্চ hit ratio-ও একটি scaling সমস্যা হতে পারে?**
Cache hit ratio হলো origin-এর বদলে cache থেকে সার্ভ করা request-এর শতাংশ। এমনকি ৯৫% hit ratio-র মানেও ৫% request এখনো database-এ পৌঁছায় — খুব উচ্চ মোট request volume-এ, সেই অবশিষ্ট ৫%-ও database যা সামলাতে পারে তার চেয়ে বেশি ট্রাফিক হতে পারে, তাই শতাংশের মতোই absolute miss rate-ও গুরুত্বপূর্ণ।

**২. cache-aside প্যাটার্ন ব্যাখ্যা করুন এবং এর একটি অসুবিধা বলুন।**
Cache-aside-এ, অ্যাপ্লিকেশন প্রথমে cache চেক করে; miss হলে এটি database query করে, তারপর ফেরত দেওয়ার আগে ফলাফলটি cache-এ লেখে। একটি অসুবিধা হলো "cold cache" সমস্যা — যেকোনো key-র প্রথম request সবসময় ধীর হয়, এবং যদি অনেক request একই সাথে একই key miss করে, তাহলে এটা database-এ একটি stampede ঘটাতে পারে।

**৩. write-back-এর বদলে কখন write-through বেছে নেবেন?**
Write-through বেছে নিন যখন read consistency অত্যন্ত গুরুত্বপূর্ণ এবং আপনি cache ও database কখনোই সিঙ্কের বাইরে থাকা সহ্য করতে পারবেন না (যেমন, financial balance), এবং আপনার write volume/latency budget synchronously দুটি সিস্টেমে লেখা সহ্য করতে পারে। Write-back বেছে নিন যখন cache crash হলে flush-না-হওয়া ডেটা হারানোর সামান্য ঝুঁকির চেয়ে write throughput এবং কম write latency বেশি গুরুত্বপূর্ণ।

**৪. write-around caching কী এবং এটা কখন কাজে লাগে?**
Write-around writeগুলো সরাসরি database-এ যায়, cache বাইপাস করে; cache শুধুমাত্র পরে একটি read-এ পূরণ করা হয় (cache-aside style-এ)। এটা write-heavy, কদাচিৎ পড়া হওয়া ডেটার জন্য কাজে লাগে যেমন log বা audit trail, যেখানে প্রতিটি write cache করা এমন ডেটার জন্য cache স্থান নষ্ট করবে যা শীঘ্রই আবার পড়া হওয়ার সম্ভাবনা কম।

**৫. দুটি cache invalidation স্ট্র্যাটেজি এবং প্রতিটির একটি ট্রেড-অফ বর্ণনা করুন।**
TTL-ভিত্তিক expiration একটি নির্দিষ্ট সময় পরে entryগুলোকে স্বয়ংক্রিয়ভাবে expire করে দেয় — সহজ এবং self-healing, কিন্তু ডেটা পুরো TTL window পর্যন্ত stale থাকতে পারে। Explicit invalidation আন্ডারলাইং ডেটা পরিবর্তন হলে সক্রিয়ভাবে cache entry সরায়/আপডেট করে — বেশি তাজা, কিন্তু প্রতিটি mutation কোড পথকে invalidate করার কথা মনে রাখতে হয়, এবং একটা মিস করলে খুঁজে বের করা কঠিন stale-data বাগ হয়।

**৬. cache stampede কী, এবং আপনি কীভাবে এটা প্রতিরোধ করবেন?**
একটি cache stampede (thundering herd) তখন ঘটে যখন একটি জনপ্রিয় cache key expire হয় এবং একই মুহূর্তে বিপুল সংখ্যক concurrent request সেটা miss করে, একসাথে database-কে আঘাত করে। প্রশমনের উপায়ের মধ্যে আছে request coalescing/locking (একটি মাত্র request cache পুনরায় পূরণ করে যখন বাকিরা অপেক্ষা করে), ব্যাকগ্রাউন্ডে রিফ্রেশ করার সময় সংক্ষিপ্তভাবে stale ডেটা সার্ভ করা, এবং TTL-কে staggered/jitter করা যাতে hot keyগুলো একসাথে expire না হয়।

**৭. LRU, LFU, এবং FIFO eviction পলিসির তুলনা করুন।**
LRU সবচেয়ে দীর্ঘ সময় ধরে অ্যাক্সেস না হওয়া entry evict করে, সম্প্রতি ব্যবহৃত ডেটাকে প্রাধান্য দেয়। LFU সবচেয়ে কম মোট অ্যাক্সেস হওয়া entry evict করে, recency-র চেয়ে ধারাবাহিকভাবে জনপ্রিয় ডেটাকে প্রাধান্য দেয়। FIFO অ্যাক্সেস প্যাটার্ন নির্বিশেষে সবচেয়ে পুরনো insert করা entry evict করে, যা সহজ কিন্তু বাস্তব-জগতের অ্যাক্সেস প্যাটার্নের জন্য প্রায়শই LRU বা LFU-এর চেয়ে কম কার্যকর।

**৮. আপনার product page cache একটি ৬০-সেকেন্ড TTL-এ সেট করা আছে, কিন্তু customer support বলছে একজন admin দাম আপডেট করার ঠিক পরেই ব্যবহারকারীরা মাঝে মাঝে পুরনো দাম দেখতে পান। আপনি কীভাবে এটা ঠিক করবেন?**
Explicit invalidation যোগ করুন: যখনই একটি দাম আপডেট হয়, শুধুমাত্র ৬০-সেকেন্ড TTL-এর ওপর নির্ভর করার বদলে সক্রিয়ভাবে সেই product-এর cache entry মুছুন বা রিফ্রেশ করুন। TTL একটি নিরাপত্তা জাল হিসেবে থাকতে পারে, কিন্তু explicit invalidation দাম-সংবেদনশীল আপডেটের জন্য staleness window দূর করে।

**৯. write-back caching কেন data loss-এর ঝুঁকি নেয়, এবং আপনি কীভাবে সেই ঝুঁকি কমাতে পারেন?**
Write-back একটি write-কে database-এ persist হওয়ার আগেই, cache-এ যাওয়া মাত্র নিশ্চিত করে দেয়; pending write flush করার আগে cache crash হলে সেই ডেটা হারিয়ে যায়। একটি durable write-ahead log বা replication দিয়ে cache-কে সমর্থন করে, ঘন ঘন/ছোট batch-এ flush করে, অথবা built-in persistence সহ একটি caching সিস্টেম ব্যবহার করে এই ঝুঁকি কমানো যায়।

**১০. আপনার cache layer সম্পূর্ণভাবে ডাউন হয়ে গেলে, আপনার অ্যাপ্লিকেশনের কী হওয়া উচিত, এবং কোন caching প্যাটার্ন এটাকে সুন্দরভাবে সামলায়?**
আদর্শভাবে অ্যাপ্লিকেশনটির gracefully degrade করা উচিত এবং সম্পূর্ণভাবে ব্যর্থ হওয়ার বদলে সরাসরি database থেকে request সার্ভ করা চালিয়ে যাওয়া উচিত (ধীর, কিন্তু কার্যকর)। Cache-aside এটাকে ভালোভাবে সমর্থন করে কারণ অ্যাপ্লিকেশন ইতিমধ্যে জানে যে cache miss হলে কীভাবে সরাসরি database query করতে হয় — এটা কেবল একটি cache outage-কে একটি স্থায়ী miss হিসেবে বিবেচনা করে।

**১১. একটি static "About Us" page-এর তুলনায় একটি "trending" বা "hot" content feed-এর জন্য আপনি কেন ইচ্ছাকৃতভাবে একটি ছোট TTL বেছে নিতে পারেন?**
একটি trending feed ঘন ঘন পরিবর্তিত হয় এবং ব্যবহারকারীরা প্রায়-বাস্তব-সময়ের freshness আশা করেন, তাই একটি ছোট TTL (সেকেন্ড) বেশিরভাগ read ট্রাফিক শোষণ করার পাশাপাশি staleness সীমিত করে। একটি static About Us page কদাচিৎ পরিবর্তিত হয়, তাই একটি দীর্ঘ TTL (ঘণ্টা বা তার বেশি) কোনো অর্থপূর্ণ freshness খরচ ছাড়াই cache দক্ষতা সর্বোচ্চ করে।

**১২. cache miss এবং stale data-এর মধ্যে পার্থক্য কী, এবং কেন এই পার্থক্যটি operationally গুরুত্বপূর্ণ?**
একটি cache miss মানে ডেটা এখনো cache-এ উপস্থিত নেই, source of truth থেকে একটি fetch দরকার। Stale data মানে cache-এ আসলে একটি entry আছে, কিন্তু সেটা আর source of truth-এর বর্তমান অবস্থা প্রতিফলিত করে না। এটা গুরুত্বপূর্ণ কারণ একটি miss শুধু latency-র খরচ করে, কিন্তু stale data সার্ভ করা এমন correctness বাগ ঘটাতে পারে যা সনাক্ত করা এবং ডিবাগ করা অনেক বেশি কঠিন।
