# অনুশীলন ও ইন্টারভিউ প্রশ্ন (Practice & Interview Questions)

**১. কেন বিভিন্ন মেশিনের wall-clock timestamp-কে একটি distributed system জুড়ে events order করার জন্য বিশ্বাস করা যায় না?**
প্রতিটি মেশিনের নিজস্ব physical clock আছে, এবং NTP synchronization থাকা সত্ত্বেও, clock-গুলো একে অপরের সাপেক্ষে drift এবং skew করে — কখনো মিলিসেকেন্ডে, network সমস্যায় কখনো আরো বেশি। যদি ভিন্ন মেশিনে দুটি events কাছাকাছি সময়ে ঘটে, তাদের wall-clock timestamp তুলনা করলে প্রকৃতপক্ষে কোনটি প্রথমে ঘটেছে সে সম্পর্কে ভুল উত্তর পাওয়া যেতে পারে, এবং পরে সেই error শনাক্ত করার কোনো উপায় থাকে না।

**২. Lamport timestamp সংজ্ঞায়িত করা দুটি নিয়ম বর্ণনা করুন।**
প্রতিটি local event process-এর নিজস্ব counter বৃদ্ধি করে। যখনই কোনো process একটি message পাঠায়, এটি তার বর্তমান counter মান অন্তর্ভুক্ত করে; গ্রহণকারী process তার নিজস্ব মান সেট করে তার বর্তমান মান এবং প্রাপ্ত মানের মধ্যে সর্বোচ্চটিতে, তারপর এটিকে এক দ্বারা বৃদ্ধি করে।

**৩. একটি Lamport timestamp আপনাকে কী নিশ্চয়তা দেয়, এবং এটি কী বলতে পারে না?**
যদি event A happened-before event B হয় (A causally B-কে প্রভাবিত করতে পারতো), তাহলে A-এর Lamport timestamp নিশ্চিতভাবে B-এর চেয়ে ছোট হবে। এটি বিপরীতটি বলতে পারে না — causal সম্পর্কহীন দুটি events (প্রকৃতপক্ষে concurrent, অসম্পর্কিত events) যেকোনো ক্রমে timestamp পেতে পারে বা কাকতালীয়ভাবে কাছাকাছি মান পেতে পারে, তাই আপনি একা Lamport timestamp দিয়ে প্রকৃত concurrency শনাক্ত করতে পারবেন না।

**৪. একটি vector clock গঠনগতভাবে একটি Lamport timestamp থেকে কীভাবে আলাদা?**
একটি Lamport timestamp একটি একক counter। একটি vector clock হলো counter-এর একটি array, system-এর প্রতিটি process-এর জন্য একটি slot — প্রতিটি process একটি local event-এ কেবল তার নিজের slot বৃদ্ধি করে, এবং একটি message পাওয়ার সময় প্রাপ্ত vector-এর সাথে element-wise maximum নেয় (তারপর নিজের slot বৃদ্ধি করে)।

**৫. দুটি vector clock ব্যবহার করে, আপনি কীভাবে নির্ধারণ করবেন দুটি events প্রকৃতপক্ষে concurrent কিনা?**
এগুলোকে element-wise তুলনা করুন। যদি vector A-এর প্রতিটি element vector B-এর সংশ্লিষ্ট element-এর চেয়ে ছোট বা সমান হয় (অন্তত একটি strictly ছোট সহ), তাহলে A happened-before B। যদি কোনো vector-ই অন্যটির উপর dominate না করে — কিছু elements A-তে বেশি, অন্যগুলো B-তে বেশি — তাহলে events প্রকৃতপক্ষে concurrent, অর্থাৎ কেউই causally অন্যটিকে প্রভাবিত করতে পারতো না।

**৬. কেন Amazon-এর Dynamo সাধারণ "last write wins" timestamp তুলনার বদলে vector clocks ব্যবহার করে?**
কারণ wall-clock timestamp-ভিত্তিক "last write wins" clock skew-এর কারণে নীরবে এবং ভুলভাবে একটি বৈধ concurrent update বাতিল করে দিতে পারে। Vector clocks Dynamo-কে সঠিকভাবে শনাক্ত করতে দেয় কখন দুটি writes প্রকৃতপক্ষে concurrent এবং সাংঘর্ষিক, যাতে conflict-টি স্পষ্টভাবে merge করা যায় বা সমাধানের জন্য উপস্থাপন করা যায়, একটি write-কে নীরবে এবং সম্ভবত ভুলভাবে overwrite করার বদলে।

**৭. Ordering-এর জন্য সীমাবদ্ধতা থাকা সত্ত্বেও physical (NTP-সিঙ্ক্রোনাইজড) wall-clock time সঠিক tool হওয়ার দুটি উদাহরণ দিন।**
এর যেকোনো দুটি: human debugging-এর জন্য logs-এ timestamps, cache TTL/expiration windows, অথবা rate-limiting time windows — এমন ক্ষেত্র যেখানে "কখন" সম্পর্কে একটি আনুমানিক, human-meaningful ধারণাই যথেষ্ট এবং সঠিকতা মেশিন জুড়ে নিখুঁত causal ordering-এর উপর নির্ভর করে না।

**৮. পরিস্থিতি: একটি distributed cache-এর cluster nodes-কে প্রতি-key expiration time নিয়ে একমত হতে হবে, যা কয়েক সেকেন্ডের মধ্যে সঠিক হতে হবে। আপনি কি এখানে physical time নাকি একটি logical/vector clock ব্যবহার করবেন, এবং কেন?**
Physical (NTP-সিঙ্ক্রোনাইজড) time — TTL expiration-এর জন্য শুধু "কখন" সম্পর্কে একটি আনুমানিক, human-meaningful ধারণা প্রয়োজন, এবং কয়েক সেকেন্ডের clock skew এই ব্যবহারের ক্ষেত্রে একটি গ্রহণযোগ্য error margin; একটি logical/vector clock এমন জটিলতা যোগ করে যা এই পরিস্থিতির প্রয়োজন নেই।

**৯. পরিস্থিতি: একটি distributed database-এর দুটি replica একে অপরের থেকে partitioned থাকা অবস্থায় একই record-এ একটি update পায়, এবং আপনাকে timestamp দিয়ে নীরবে একটি বেছে নেওয়ার বদলে সঠিকভাবে এটিকে একটি conflict হিসেবে শনাক্ত করতে হবে। এই ভিডিও থেকে কোন tool প্রযোজ্য, এবং কেন?**
Vector clocks — দুটি updates-এর vector clock তুলনা করলে দেখা যাবে কেউই অন্যটিকে dominate করছে না, সঠিকভাবে এগুলোকে concurrent, সাংঘর্ষিক writes হিসেবে শনাক্ত করে যাদের স্পষ্ট merge/resolution logic প্রয়োজন, clock skew যাকে অনির্ভরযোগ্য করে তুলতে পারে এমন wall-clock timestamp বিশ্বাস করার বদলে।

**১০. সত্য বা মিথ্যা: একটি vector clock সবসময় আপনাকে বলতে পারে দুটি events-এর মধ্যে কোনটি real, physical time-এ "প্রথমে" ঘটেছে।**
মিথ্যা। একটি vector clock causal ordering (happened-before) সম্পর্কে বলে এবং concurrency শনাক্ত করতে পারে, কিন্তু এটি physical/wall-clock time সম্পর্কে কিছু বলে না — দুটি concurrent events (vector clock অনুযায়ী) প্রকৃতপক্ষে ভিন্ন real-world মুহূর্তে ঘটে থাকতে পারে; একটি vector clock-এর মূল বিষয়টি হলো "কোনটি physical time-এ প্রথমে ঘটেছে" প্রায়ই সঠিকতার উদ্দেশ্যে অর্থবহ বা উত্তরযোগ্য প্রশ্নই নয়।
</content>
