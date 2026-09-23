# ফলো-আপ ইন্টারভিউ প্রশ্ন

মূল ডিজাইনের পর গভীরতা পরীক্ষা করতে এগুলো ব্যবহার করুন। Model answer সংক্ষিপ্ত — একটি বাস্তব ইন্টারভিউতে মৌখিকভাবে বিস্তারিত করুন।

**১. ৫ কোটি follower-বিশিষ্ট একজন celebrity account আপনি কীভাবে সামলাবেন?**
সেই account-এর জন্য fan-out-on-write স্কিপ করুন। Follower সংখ্যা একটি threshold অতিক্রম করলে তা শনাক্ত করুন, এবং তার বদলে তাদের post fan-out-on-read দিয়ে সার্ভ করুন: followers read সময়ে celebrity-র সাম্প্রতিক post নিজেদের precomputed timeline-এ merge করে। এতে প্রতিটি post-এ ৫ কোটি-write burst এড়ানো যায় এবং cache layer-এ hot-key/hot-shard সমস্যা প্রতিরোধ হয়।

**২. পুরোপুরি chronological ক্রমে না দেখিয়ে আপনি কীভাবে feed item rank করবেন?**
প্রতিটি candidate post-কে একটি model দিয়ে স্কোর করুন, signal হিসেবে ব্যবহার করে recency, author affinity (এই author-এর সাথে ব্যবহারকারীর engagement কত ঘন ঘন), predicted engagement (likes/comments/shares-এর সম্ভাবনা), এবং content type। Fan-out সময়ে একটি rough score precompute করুন যাতে তা sorted-set cache-এ স্কোর অনুযায়ী insert করা যায়, তারপর ঐচ্ছিকভাবে latency-sensitive personalization-এর জন্য read সময়ে top N candidate-কে সতেজ signal দিয়ে re-rank করুন।

**৩. ১০,০০০ account follow করেন এমন একজন ব্যবহারকারীকে আপনি কীভাবে সামলাবেন?**
এদের বেশিরভাগ account-এর জন্য fan-out-on-write থাকলেও, হাজার হাজার contribution পড়া ও merge করা ব্যবস্থাপনাযোগ্য, কারণ writes ইতিমধ্যেই তাদের cached timeline pre-populate করে রেখেছে — read তখনও একটি single cache fetch। বেশি cost পড়ে write দিকে: এই ১০,০০০ followee-এর প্রত্যেকেই এই ব্যবহারকারীর কাছে fan out করে, তাই এই ব্যবহারকারীর timeline update rate বেশি; cached timeline-এর দৈর্ঘ্য সীমিত রাখুন (যেমন, সর্বশেষ ~৮০০টি এন্ট্রি) এবং memory বাউন্ড রাখতে eviction ও miss-এ rebuild-এর উপর নির্ভর করুন।

**৪. একেবারে নতুন একজন ব্যবহারকারীর feed আপনি কীভাবে backfill করবেন, যখন তার এখনো কোনো cached timeline নেই?**
প্রথম login-এ (cache miss), fan-out-on-read পথে fall back করুন: follow graph query করে দেখুন সে কাকে follow করে, sharded post store থেকে প্রতিটি followee-এর সাম্প্রতিক post টেনে আনুন, merge ও rank করুন, ফলাফল ফেরত দিন, এবং asynchronous-ভাবে cache পূরণ করুন যাতে পরবর্তী reads দ্রুত হয়।

**৫. একটি deleted বা edited post আপনি কীভাবে সেই feed-গুলোতে propagate করবেন যেখানে এটি ইতিমধ্যে fan out হয়ে গেছে?**
প্রতিটি timeline entry-তে content ডুপ্লিকেট করার বদলে post-কে ID দিয়ে সংরক্ষণ করুন এবং read সময়ে পুরো content fetch করুন (hydration) — cache post ID সংরক্ষণ করে, পুরো post body নয়। Delete-এর ক্ষেত্রে, sharded post store-এ post-টিকে deleted হিসেবে চিহ্নিত করুন (অথবা সরিয়ে দিন) এবং Feed Service-কে hydration-এর সময় অনুপস্থিত/deleted ID filter out বা refresh করতে দিন। Edit-এর ক্ষেত্রে, post store-ই source of truth, তাই cached ID reference স্বয়ংক্রিয়ভাবে আপডেট হওয়া content দেখায় — কোনো fan-out re-push দরকার নেই।

**৬. Ranking update real time-এ (stream) নাকি batch-এ compute করা উচিত?**
Batch বনাম stream trade-off রেফারেন্স করে একটি hybrid ব্যবহার করুন: হালকা, latency-sensitive signal (recency, তাৎক্ষণিক engagement count) সতেজতার জন্য একটি stream processing pipeline দিয়ে আপডেট করা হয়; ভারী signal (দীর্ঘমেয়াদী affinity model, engagement-prediction model-এর weight) পর্যায়ক্রমিক batch job দিয়ে retrain ও refresh করা হয়, কারণ এগুলো ধীরে পরিবর্তিত হয় এবং অনেক বেশি compute-intensive।

**৭. Celebrity account-এর কোটি কোটি follower থাকলে আপনি কীভাবে follow-graph store-কে performant রাখবেন?**
Follow-graph store-কে post store থেকে আলাদাভাবে shard করুন, কারণ access pattern ভিন্ন (graph traversal বনাম key-value post lookup)। Celebrity account-এর জন্য, তাদের পুরো follower list একটি single fan-out job-এ কখনো লোড করা এড়িয়ে চলুন; বরং তা queue-এর মাধ্যমে batch-এ paginate/stream করুন, অথবা সেই account-গুলোর জন্য fan-out-on-read ব্যবহার করে তা লোড করাই এড়িয়ে যান।

**৮. Fan-out Service মাঝপথে পিছিয়ে পড়লে বা crash করলে কী হবে?**
যেহেতু Post Service synchronous-ভাবে fan out করার বদলে একটি durable message queue-তে (Kafka) publish করে, post-টি নিজেই fan-out progress নির্বিশেষে নিরাপদে persist থাকে। Fan-out Service তার সর্বশেষ committed offset থেকে পুনরায় শুরু করতে পারে, এবং consumer-দের horizontally scale করে ধরে ফেলা যায়। Followers সংক্ষিপ্তভাবে একটি বিলম্বিত post দেখতে পারে — আমাদের eventual consistency requirement-এর অধীনে যা গ্রহণযোগ্য।

**৯. একটি trending celebrity post-এর জন্য একটি একক Redis node bottleneck হয়ে ওঠা আপনি কীভাবে এড়াবেন?**
Redis cluster জুড়ে key সমানভাবে ছড়িয়ে দিতে consistent hashing ব্যবহার করুন, তারপর সেই নির্দিষ্ট hot key-এর জন্য targeted মিটিগেশন যোগ করুন: সেই shard-এর জন্য read replica, তার সামনে short-TTL local/edge caching, অথবা request coalescing যাতে একই key-এর জন্য একসাথে আসা অনেক read একটি single backend fetch-এ একত্র হয়ে যায়।

**১০. Ranking algorithm-এর পরিবর্তন engagement-এ regression ঘটায়নি তা আপনি কীভাবে পরীক্ষা করবেন?**
A/B testing-এর মাধ্যমে rollout করুন: একটি নির্দিষ্ট শতাংশ ব্যবহারকারীকে নতুন ranking model-এ route করুন, control group-এর সাথে engagement metric (time spent, likes, shares, session return rate) তুলনা করুন, এবং treatment group যদি অন্য metric যেমন content diversity ক্ষতিগ্রস্ত না করে পরিসংখ্যানগতভাবে উল্লেখযোগ্য উন্নতি দেখায় তবেই তা ধীরে ধীরে বাড়ান।

**১১. পুরো feed pipeline ডুপ্লিকেট না করে আপনি কীভাবে real-time notification (যেমন, "X liked your post") সমর্থন করবেন?**
বিদ্যমান pub-sub backbone পুনরায় ব্যবহার করুন: একই বা একটি সমান্তরাল Kafka topic-এ interaction-এর জন্য একটি হালকা event emit করুন, এবং একটি আলাদা Notification Service সেটিতে subscribe করুক — Fan-out Service থেকে decoupled, কারণ notification-এর delivery guarantee (সাধারণত at-least-once, low latency) feed-এর precomputed timeline থেকে ভিন্ন।

**১২. ব্যবহারকারীর সংখ্যা দ্বিগুণ হলে আপনি কীভাবে post store ও cache স্কেল করবেন?**
sharded post store এবং Redis cache cluster — উভয়েই আরো shard/node যোগ করুন; যেহেতু উভয়ই consistent hashing ব্যবহার করে, নতুন node-এ কেবল key-এর একটি ছোট অংশই সরাতে হয়, পুরো remap নয়, যা স্কেল-আউটের সময় cache-miss storm এবং rebalancing cost কমিয়ে দেয়।
