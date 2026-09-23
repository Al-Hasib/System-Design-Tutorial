# অনুশীলন ও Interview প্রশ্ন

**১. Sharding এবং partitioning-এর মধ্যে পার্থক্য কী?**
Partitioning হলো ডেটাকে ছোট অংশে ভাগ করার সাধারণ ধারণা, এবং এটা vertical (column/table অনুযায়ী) বা horizontal (row অনুযায়ী) হতে পারে। Sharding নির্দিষ্টভাবে horizontal partitioning বোঝায় যেখানে ফলাফল অংশগুলো ("shard") আলাদা, স্বাধীন ডেটাবেস instance-এ থাকে। তাই সব sharding-ই partitioning, কিন্তু সব partitioning sharding নয়।

**২. কেন শুধু replication write scaling বা dataset আকারের সমস্যা সমাধান করে না?**
বেশিরভাগ replication সেটআপে (যেমন, master-slave), প্রতিটি write-কে এখনও একটি single primary node দিয়ে যেতে হয়, তাই আরও replica যোগ করলে write capacity বাড়ে না। এছাড়াও, প্রতিটি replica সাধারণত সম্পূর্ণ dataset-এর একটি পূর্ণ কপি সংরক্ষণ করে, তাই replication কোনো single node-কে কত ডেটা ধরে রাখতে হবে তা কমায় না — এটা শুধু duplicate করে।

**৩. Range-based sharding এবং এর প্রধান দুর্বলতা ব্যাখ্যা করুন।**
Range-based sharding shard key-এর contiguous range প্রতিটি shard-কে বরাদ্দ করে (যেমন, ID ১–১০ লাখ shard 1-এ, ১০ লাখ–২০ লাখ shard 2-তে)। এটা দক্ষ range query সমর্থন করে কারণ একটি range scan সাধারণত শুধু এক বা কয়েকটি shard স্পর্শ করে। এর প্রধান দুর্বলতা হলো hotspotting: key যদি সময়ের সাথে সম্পর্কিত হয় বা sequential ভাবে বাড়ে, তাহলে নতুন/সাম্প্রতিক ডেটা — এবং প্রায়শই সবচেয়ে active ডেটা — একটি shard-এ জমা হয় যখন পুরনো shard-গুলো তুলনামূলক অলস বসে থাকে।

**৪. Hash-based sharding এবং এর প্রধান দুর্বলতা ব্যাখ্যা করুন।**
Hash-based sharding shard key-কে একটি hash function-এর মধ্য দিয়ে চালায় (প্রায়ই shard-সংখ্যা দিয়ে modulo-র সাথে মিলিয়ে) একটি row কোন shard-এ থাকবে তা ঠিক করতে। এটা ডেটা এবং load খুব সমানভাবে ছড়িয়ে দেয় কারণ একটি ভালো hash function key-গুলোকে pseudo-randomly বণ্টন করে। দুর্বলতা হলো range query ব্যয়বহুল হয়ে যায় — sequential key-গুলো shard জুড়ে ছড়িয়ে যায়, তাই একটি range query-কে অবশ্যই প্রতিটি shard-এ fan out করতে হবে এবং ফলাফল merge করতে হবে।

**৫. Directory-based sharding কী, এবং কখন আপনি range বা hash sharding-এর বদলে এটি বেছে নেবেন?**
Directory-based sharding একটি formula দিয়ে mapping হিসাব করার বদলে একটি explicit lookup service ব্যবহার করে যা প্রতিটি key (বা key range)-কে একটি নির্দিষ্ট shard-এর সাথে map করে। আপনি এটি বেছে নেবেন যখন আপনার সর্বোচ্চ flexibility দরকার — individual key-গুলো shard-এর মধ্যে সরানোর, অসমভাবে loaded shard-গুলোকে সাথে সাথে rebalance করার, বা heterogeneous shard capacity সমর্থন করার ক্ষমতা — এবং প্রতিটি query-তে directory-র সাথে পরামর্শ করার বাড়তি complexity এবং অতিরিক্ত network hop সহ্য করতে পারেন।

**৬. Consistent hashing কী, এবং sharding-এর সাথে এটি কেন প্রাসঙ্গিক?**
Consistent hashing এমন একটি hashing টেকনিক যা shard যোগ বা বাদ দেওয়ার সময় কতগুলো key সরাতে হবে তা কমিয়ে দেয়, naive `hash(key) % N`-এর বিপরীতে, যেখানে N পরিবর্তন করলে প্রায় প্রতিটি key remap হয়ে যায়। এটা প্রাসঙ্গিক কারণ resharding — বৃদ্ধির সাথে সাথে capacity যোগ করা — একটি sharded সিস্টেম চালানোর সবচেয়ে কষ্টকর operational অংশগুলোর একটি, এবং consistent hashing সেই প্রক্রিয়াকে অনেক কম বিঘ্নকর করে তোলে (সাধারণত মাত্র প্রায় `1/N` key সরে)।

**৭. একটি ভালো shard key কীসে তৈরি হয়? কী এড়িয়ে চলা উচিত?**
একটি ভালো shard key-তে উচ্চ cardinality থাকে (অনেক distinct value যাতে ডেটা shard জুড়ে ছড়িয়ে পড়ে), এটা আপনার প্রধান query pattern-এর সাথে মিলে যায় (যাতে বেশিরভাগ query একটি single shard-এ route করা যায়), এবং অল্প সংখ্যক value-তে traffic কেন্দ্রীভূত হওয়া এড়ায়। যেসব key sequential/time-correlated (সবচেয়ে নতুন shard-এ hotspot তৈরি করে) বা যাদের বাস্তব জগতে skewed distribution আছে, যেমন একজন "celebrity" ব্যবহারকারী বা একটি একক প্রধান tenant যে অসামঞ্জস্যপূর্ণ traffic পায়, সেগুলো এড়িয়ে চলুন।

**৮. Sharding এমন কী চ্যালেঞ্জ নিয়ে আসে যা একটি single-database সিস্টেমের নেই?**
Cross-shard join এবং transaction কঠিন হয়ে যায়, কারণ shard জুড়ে সম্পর্কিত ডেটা distributed transaction coordination (যেমন, two-phase commit বা saga) ছাড়া join বা atomically commit করা যায় না। Shard অসমভাবে বাড়ার সাথে সাথে ডেটা rebalance করা operational ভাবে কঠিন। এবং সামগ্রিক operational complexity বাড়ে — আপনি এখন একটির বদলে N-টি ডেটাবেস জুড়ে monitor, backup, এবং schema migrate করেন।

**৯. ৫০ কোটি row-এর একটি table, যা একটি single Postgres instance-এর চেয়ে বড় হয়ে যাচ্ছে, তা আপনি কীভাবে shard করবেন?**
প্রথমে, প্রধান access pattern চিহ্নিত করুন — যেমন, যদি বেশিরভাগ query `customer_id` বা `tenant_id` দিয়ে filter করে, তাহলে সেটাই একটি শক্তিশালী shard-key candidate। সেই key-তে একটি hash-based scheme বেছে নিন (Citus-এর মতো একটি extension দিয়ে, বা manually) সমান বণ্টন পাওয়ার জন্য এবং hotspot এড়াতে, growth-এর জন্য যথেষ্ট জায়গা রেখে যথেষ্ট shard ব্যবহার করে। Migration-কে একটি single cutover-এর বদলে একটি online, incremental data move (dual-write বা CDC-based backfill) হিসেবে পরিকল্পনা করুন, এবং application/router layer-কে এমনভাবে design করুন যাতে single-tenant query একটি shard-এ পরিচালিত হয় আর cross-tenant analytics-এর জন্য scatter-gather দরকার হবে তা মেনে নেয়।

**১০. একটি multi-tenant SaaS application-এর জন্য আপনি কোন shard key বেছে নেবেন, এবং কেন?**
`tenant_id` (বা `organization_id`) সাধারণত সেরা পছন্দ, কারণ একটি multi-tenant সিস্টেমে প্রায় প্রতিটি query ইতিমধ্যেই একটি single tenant-এর মধ্যে scoped থাকে — তাদের user, record, এবং setting টেনে আনা। `tenant_id` দিয়ে shard করার মানে হলো অধিকাংশ query ঠিক একটি shard-এ গিয়ে পড়ে, latency কম রাখে এবং cross-shard join এড়ায়, যখন ডেটা স্বাভাবিকভাবে প্রতি-customer isolate হয়, যা compliance এবং noisy-neighbor containment-ও সহজ করে (ধরে নেওয়া যে কোনো একক tenant অন্যদের তুলনায় অসামঞ্জস্যপূর্ণভাবে বিশাল নয়)।

**১১. আপনার hash-sharded সিস্টেমকে "সব customer জুড়ে গত সপ্তাহে মোট কতগুলো order দেওয়া হয়েছে" এই প্রশ্নের উত্তর দিতে হবে। আপনি এটা দক্ষভাবে কীভাবে সামলাবেন?**
যেহেতু shard key এই cross-cutting analytical query-এর সাথে মেলে না, আপনি সাধারণত query-কে সব shard-এ সমান্তরালভাবে fan out করবেন (scatter-gather) এবং application layer-এ partial count একত্র করবেন। ঘন ঘন প্রয়োজনীয় aggregate-এর জন্য, একটি ভালো দীর্ঘমেয়াদী পদ্ধতি হলো একটি change-data-capture pipeline দ্বারা পরিচালিত একটি আলাদা analytics store (যেমন, একটি data warehouse বা pre-aggregated rollup table) বজায় রাখা, যাতে প্রতিটি reporting query-র জন্য প্রতিটি operational shard-এ আঘাত করতে না হয়।

**১২. Range-based, naive hash-based, এবং consistent-hash-based sharding-এর resharding-এর কঠিনতা তুলনা করুন।**
Range-based sharding প্রায়ই একটি range-কে দুই ভাগে ভাগ করে rebalance করা যায়, যার জন্য শুধু split range-এর ডেটা সরাতে হয়। Naive hash-based sharding (`hash % N`) সবচেয়ে খারাপ ক্ষেত্র: N পরিবর্তন করলে বেশিরভাগ key remap হয়ে যায়, যা একটি বিশাল data migration বাধ্য করে। Consistent hashing hash-based পদ্ধতির মধ্যেই এই নির্দিষ্ট সমস্যাটি সমাধান করে — একটি node যোগ বা বাদ দিলে মাত্র প্রায় `1/N` key পুনরায় বরাদ্দ হয়, যা এটাকে hash-sharded সিস্টেমগুলোর জন্য পছন্দের টেকনিক করে তোলে যাদের সময়ের সাথে shard-সংখ্যা scale করতে হবে।
