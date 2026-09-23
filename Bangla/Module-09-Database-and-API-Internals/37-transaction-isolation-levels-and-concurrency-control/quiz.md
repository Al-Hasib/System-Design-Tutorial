# Practice & Interview Questions

**১. Dirty read কী, এবং কোন isolation level এটির অনুমতি দেয়?**
একটি dirty read তখন ঘটে যখন একটি transaction অন্য একটি transaction-এর uncommitted পরিবর্তন read করে। সেই অন্য transaction পরে rollback করলে, reader-টি এমন ডেটার ওপর কাজ করে ফেলেছে যা আসলে কখনো ছিলই না। শুধুমাত্র Read Uncommitted dirty read-এর অনুমতি দেয়; Read Committed এবং তার ওপরের level-গুলো এটি প্রতিরোধ করে।

**২. একটি non-repeatable read এবং একটি phantom read-এর মধ্যে পার্থক্য ব্যাখ্যা করুন।**
একটি non-repeatable read হলো একটি transaction-এর মধ্যে একই row দুইবার read করে ভিন্ন value পাওয়া, কারণ মাঝখানে অন্য একটি transaction সেই নির্দিষ্ট row-তে একটি পরিবর্তন commit করেছে। একটি phantom read হলো একই query (একটি single row নয়) পুনরায় চালিয়ে ভিন্ন সেট মিলে যাওয়া row পাওয়া, কারণ মাঝখানে অন্য একটি transaction সেই শর্তের সাথে মিলে যাওয়া row insert বা delete করেছে।

**৩. Serializable বেশি anomaly প্রতিরোধ করা সত্ত্বেও কেন একটি সিস্টেম Serializable-এর বদলে Read Committed বেছে নেবে?**
Serializable isolation-এর জন্য অনেক বেশি coordination প্রয়োজন — ব্যাপক locking অথবা conflict detection — যা throughput কমায় এবং abort/retry হওয়া transaction-এর হার বাড়ায়। Read Committed হলো একটি বাস্তবসম্মত default যা dirty read (সবচেয়ে বিপজ্জনক anomaly) প্রতিরোধ করে, একই সাথে সেই workload-গুলোর জন্য অনেক বেশি concurrency বজায় রাখে যেগুলো বাকি, বিরল anomaly-গুলো সহ্য করতে পারে।

**৪. Pessimistic locking এবং optimistic concurrency control-এর তুলনা করুন, এবং এমন একটি করে পরিস্থিতির বর্ণনা দিন যেখানে প্রতিটি ভালো পছন্দ।**
Pessimistic locking ডেটা স্পর্শ করার আগেই একটি lock নেয়, contention এবং সম্ভাব্য deadlock-এর খরচে আগে থেকেই conflict প্রতিরোধ করে — একটি সীমিত inventory count-এর মতো hot, high-contention row-এর জন্য এটি ভালো মানানসই। Optimistic concurrency control transaction-গুলোকে lock ছাড়াই এগিয়ে যেতে দেয় এবং শুধু commit-এর সময় conflict চেক করে, conflict ঘটলে retry করে — সাধারণ low-contention ডেটার জন্য এটি ভালো মানানসই, যেখানে conflict বিরল এবং locking overhead বেশিরভাগ সময় অপচয় হতো।

**৫. MVCC এমন কোন সমস্যা সমাধান করে যা pure locking করে না?**
Pure locking reader-দের writer-দের পেছনে block হতে বাধ্য করতে পারে (এবং উল্টোটাও), এমনকি যখন একটি reader শুধু একটি consistent snapshot চায়, কিছু পরিবর্তন করতে নয়। MVCC প্রতিটি row-এর একাধিক version রাখে যাতে একটি reader কখনো একটি concurrent writer-এর জন্য block না হয়েই একটি consistent, committed snapshot দেখতে পারে, এবং একটি writer-কে কখনো একটি concurrent reader শেষ হওয়ার জন্য অপেক্ষা করতে হয় না।

**৬. PostgreSQL-এর মতো একটি MVCC-ভিত্তিক database-এর কেন `VACUUM`-এর মতো একটি প্রসেস প্রয়োজন?**
কারণ MVCC পুরোনো row version ধরে রাখে যাতে একটি পরিবর্তন commit হওয়ার আগে শুরু হওয়া transaction-গুলো তখনও তাদের নিজস্ব শুরুর সময়ের সাথে সামঞ্জস্যপূর্ণ version দেখতে পারে। যখন আর কোনো active transaction-এর একটি পুরোনো version-এর প্রয়োজন থাকে না, তখন সেটি মৃত জায়গায় পরিণত হয় যা পুনরুদ্ধার করতে হয় — `VACUUM` হলো PostgreSQL-এর background প্রসেস যা এই পুরোনো version-গুলো পরিষ্কার করে এবং table bloat প্রতিরোধ করে।

**৭. পরিস্থিতি: দুজন গ্রাহক ঠিক একই মুহূর্তে একটি ফ্লাইটের শেষ available সিটটি বুক করার চেষ্টা করছে। আপনি কি শুধু আপনার database-এর default isolation level এবং MVCC-এর ওপর নির্ভর করবেন, নাকি আরও কিছু যোগ করবেন? কেন?**
এই ধরনের একটি high-stakes, ভুলের জন্য কম সহনশীলতার পরিস্থিতির জন্য, শুধুমাত্র MVCC-এর snapshot isolation-এর ওপর নির্ভর করলে উভয় transaction-ই তাদের নিজস্ব consistent snapshot-এর ভিত্তিতে একটি সিট available আছে বলে বিশ্বাস করার ঝুঁকি থাকে। এতটা গুরুত্বপূর্ণ কিছুর জন্য optimistic retry logic-কে বিশ্বাস করার বদলে, সেই নির্দিষ্ট row-তে স্পষ্টভাবে pessimistic locking জোর করা নিরাপদ (যেমন, `SELECT ... FOR UPDATE`) যাতে দ্বিতীয় transaction-টি প্রথমটি commit হওয়ার পর অপেক্ষা করে availability পুনরায় check করতে বাধ্য হয়।

**৮. Deadlock কী, এবং একটি database সাধারণত এটি কীভাবে সামলায়?**
একটি deadlock তখন ঘটে যখন দুই বা ততোধিক transaction প্রত্যেকেই এমন একটি lock ধরে রাখে যা অন্যটির প্রয়োজন, তাই কেউই এগোতে পারে না। Database-গুলো এই অবস্থা সনাক্ত করে (প্রায়ই একটি wait-for graph-এর মাধ্যমে) এবং একটি transaction-কে ("victim") abort করে এটি সমাধান করে, যা তার lock-গুলো ছেড়ে দেয় যাতে অন্যটি এগিয়ে যেতে পারে; aborted transaction-এর application কোডকে retry করার জন্য প্রত্যাশা করা হয়।

**৯. সত্য নাকি মিথ্যা: Repeatable Read-এর অধীনে, SQL স্ট্যান্ডার্ড অনুযায়ী phantom read অসম্ভব হওয়ার নিশ্চয়তা আছে।**
মিথ্যা। SQL স্ট্যান্ডার্ডের Repeatable Read-এর সংজ্ঞা শুধুমাত্র নিশ্চিত করে যে পৃথকভাবে read করা row-গুলোতে কোনো non-repeatable read হবে না — phantom read (একটি range শর্তের সাথে মিলে যাওয়া নতুন row) স্ট্যান্ডার্ড অনুযায়ী এই level-এ প্রযুক্তিগতভাবে এখনও অনুমোদিত, যদিও কিছু database engine-এর প্রকৃত implementation (যেমন MySQL-এর InnoDB) gap locking-এর মতো অতিরিক্ত mechanism-এর মাধ্যমে অনেক phantom পরিস্থিতি প্রতিরোধ করে।

**১০. Optimistic concurrency control-এর জন্য কেন application layer-এ retry logic implement করা প্রয়োজন, অথচ pessimistic locking-এর সাধারণত তা প্রয়োজন হয় না?**
OCC-তে, একটি conflict শুধুমাত্র commit-এর সময় আবিষ্কৃত হয়, এবং transaction-টি স্বয়ংক্রিয়ভাবে সমাধান হওয়ার বদলে abort হয়ে যায় — application-কে এই ব্যর্থতা সনাক্ত করতে হয় এবং সিদ্ধান্ত নিতে হয় সম্পূর্ণ operation-টি retry করবে কিনা/কীভাবে করবে। Pessimistic locking-এর ক্ষেত্রে, database নিজেই conflicting transaction-গুলোকে lock মুক্ত না হওয়া পর্যন্ত অপেক্ষা করায়, তাই application-কে স্পষ্টভাবে retry করার প্রয়োজন ছাড়াই operation-টি স্বাভাবিকভাবে ক্রমানুসারে এগিয়ে যায়।
