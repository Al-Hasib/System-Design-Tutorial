# Practice ও Interview প্রশ্ন

**১. প্রতিটি ACID property একটি এক-লাইনের উদাহরণসহ ব্যাখ্যা করুন।**
- Atomicity: একটি ব্যাংক ট্রান্সফারে, হয় debit এবং credit দুটোই ঘটে, নয়তো কোনোটিই ঘটে না।
- Consistency: একটি transaction একটি অ্যাকাউন্ট ব্যালেন্স ঋণাত্মক রাখতে পারে না যদি একটি constraint তা নিষিদ্ধ করে।
- Isolation: শেষ এয়ারলাইন সিটের জন্য দুটি একযোগে সিট বুকিং উভয়েই সফল হবে না।
- Durability: একবার একটি transaction commit হয়ে গেলে, এটি ঠিক পরেই একটি সার্ভার ক্র্যাশের পরেও টিকে থাকে, কারণ এটি একটি durable log-এ লেখা হয়েছিল।

**২. BASE-এর প্রতিটি অক্ষর কী বোঝায়, এবং এটি ACID থেকে কীভাবে আলাদা?**
Basically Available, Soft state, Eventual consistency। ACID-এর বিপরীতে, যা correctness নিশ্চিত না হওয়া পর্যন্ত একটি response block/delay করে, BASE সবসময় দ্রুত একটি response রিটার্ন করে, replica ক্যাচ-আপ করার সাথে সাথে ব্যাকগ্রাউন্ডে পরিবর্তিত হওয়া state সহ্য করে, এবং শুধু নিশ্চয়তা দেয় যে replica গুলো *অবশেষে* converge করবে, তাৎক্ষণিকভাবে নয়।

**৩. ACID এবং BASE কীভাবে CAP theorem-এর সাথে সম্পর্কিত?**
CP-leaning সিস্টেম (partition-এর সময় availability-র চেয়ে consistency প্রাধান্য দেয়) সাধারণত ACID guarantee প্রদান করে, কারণ তারা correctness-কে প্রাধান্য দেয় এমনকি যদি তার মানে request block বা reject করা হয়। AP-leaning সিস্টেম (consistency-র চেয়ে availability প্রাধান্য দেয়) সাধারণত BASE guarantee প্রদান করে, সবসময় সাড়া দেওয়ার বিনিময়ে সাময়িক staleness গ্রহণ করে।

**৪. চারটি SQL standard isolation level-এর নাম বলুন এবং দুর্বলতম থেকে শক্তিশালীতম পর্যন্ত সাজান।**
Read Uncommitted (দুর্বলতম, dirty read সম্ভব) → Read Committed → Repeatable Read → Serializable (সবচেয়ে শক্তিশালী, transaction গুলো sequentially চলার মতো আচরণ করে)। শক্তিশালী isolation anomaly কমায় কিন্তু locking/contention বাড়ায় এবং throughput কমায়।

**৫. ACID-এ "Consistency" এবং CAP-এ "Consistency"-র মধ্যে পার্থক্য কী?**
ACID-এর Consistency মানে একটি transaction database-কে এক valid state থেকে আরেক valid state-এ নিয়ে যায়, schema constraint এবং নিয়ম মেনে। CAP-এর Consistency মানে একটি distributed system-এর সব নোড একই সময়ে একই, সর্বশেষ data রিটার্ন করে। এগুলো আত্মায় সম্পর্কিত কিন্তু ভিন্ন পরিসর বর্ণনা করে — একটি প্রতি-transaction data validity, অন্যটি cross-node data agreement।

**৬. একটি read-heavy analytics dashboard-এর জন্য আপনি কি schema normalize করবেন নাকি denormalize করবেন, এবং কেন?**
Denormalize করব। Analytics dashboard গুলো সাধারণত বিশাল পরিমাণ data-এর উপর aggregate query চালায় এবং storage efficiency বা write simplicity-র চেয়ে দ্রুত read-কে প্রাধান্য দেয়; data আগে থেকে join করা বা flatten করা (যেমন, wide table বা embedded document-এ) query time-এ ব্যয়বহুল join এড়ায়, এবং data প্রায়ই লেখার চেয়ে অনেক বেশি বার পড়া হয়।

**৭. একটি "update anomaly"-এর উদাহরণ দিন যা normalization প্রতিরোধ করে।**
যদি একজন গ্রাহকের ঠিকানা Users table-এ একবার সংরক্ষণ করার বদলে এক হাজার অর্ডার row-এ নকল করা হয়, তাহলে তাদের ঠিকানা আপডেট করার মানে হয় হাজার হাজার row আপডেট করা (ত্রুটিপ্রবণ), নয়তো কিছু row stale রেখে দেওয়া, যা অসামঞ্জস্যপূর্ণ data তৈরি করে। Normalization ঠিকানাটি একবার সংরক্ষণ করে, তাই একটি আপডেট সবখানে ঠিক করে দেয়।

**৮. Distributed NoSQL database গুলো কেন প্রায়ই ACID-এর চেয়ে BASE-কে প্রাধান্য দেয়?**
অনেকগুলো ভৌগোলিকভাবে বিতরণ করা নোড জুড়ে কঠোর ACID-স্টাইল consistency এবং isolation প্রয়োগ করতে ভারী coordination প্রয়োজন (যেমন, distributed lock বা consensus protocol), যা latency যোগ করে এবং network সমস্যার সময় availability কমাতে পারে। BASE সাময়িক staleness অনুমতি দিয়ে সেই coordination খরচ এড়িয়ে যায়, যা horizontal scale করতে এবং বিশ্বব্যাপী available থাকতে হবে এমন সিস্টেমের জন্য বেশি উপযুক্ত।

**৯. আক্রমণাত্মক denormalization-এর একটি ব্যবহারিক অসুবিধা কী?**
Write complexity বেড়ে যায় কারণ একটি একক logical আপডেটের জন্য একই data-র একাধিক নকল কপি আপডেট করতে হতে পারে। এটি একটি consistency risk-ও তৈরি করে — যদি একটি কপি মিস হয় বা দেরি হয়, ভিন্ন read গুলো একই তথ্যের ভিন্ন, পরস্পরবিরোধী সংস্করণ রিটার্ন করতে পারে।

**১০. 1NF, 2NF, এবং 3NF কনসেপ্চুয়াল লেভেলে বর্ণনা করুন।**
1NF: প্রতিটি column একটি একক, atomic মান ধারণ করে (একটি field-এ কোনো list বা repeating group নেই)। 2NF: প্রতিটি non-key column সম্পূর্ণ primary key-এর উপর নির্ভরশীল, একটি composite key-এর শুধু একটি অংশের উপর নয়। 3NF: প্রতিটি non-key column শুধুমাত্র key-এর উপর নির্ভরশীল, অন্য কোনো non-key column-এর উপর নয় (কোনো transitive dependency নেই)।

**১১. একটি social media platform post-এ একটি "like count" দেখায় যা মাঝে মাঝে বিভিন্ন ব্যবহারকারীর মধ্যে কয়েক সেকেন্ড lag করে। এটা কি একটা bug? ব্যাখ্যা করুন।**
এটা অগত্যা কোনো bug নয় — এটা বিশাল স্কেল এবং availability সমর্থন করার জন্য একটি eventually consistent (BASE) architecture বেছে নেওয়ার স্বাভাবিক ফলাফল। যতক্ষণ পর্যন্ত write বন্ধ হওয়ার শীঘ্রই count সঠিক মানে converge করে, ততক্ষণ একটি non-critical, উচ্চ-ভলিউম counter-এ সংক্ষিপ্ত staleness speed এবং uptime-এর জন্য একটি গ্রহণযোগ্য trade-off।

**১২. আপনি যদি একটি core banking ledger ডিজাইন করতেন, তাহলে আপনি ACID + normalized নাকি BASE + denormalized বেছে নেবেন? আপনার উত্তর যুক্তিসহ ব্যাখ্যা করুন।**
ACID + normalized। Financial ledger-এর কঠোর correctness প্রয়োজন — কোনো হারানো বা duplicate transaction নয়, কোনো stale balance নয় যা overdraft বা double-spending-এর অনুমতি দিতে পারে — এবং normalization নিশ্চিত করে প্রতিটি balance এবং account fact একবার সংরক্ষিত থাকে, inconsistency এড়িয়ে। একটি ভুল financial transaction-এর উচ্চ খরচের তুলনায় অতিরিক্ত coordination এবং সামান্য বেশি latency-র খরচ পুরোপুরি মূল্যবান।
