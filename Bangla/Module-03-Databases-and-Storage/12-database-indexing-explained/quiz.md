# Practice & Interview Questions

**১. একটি database index কী, এবং এটি কেন query-কে দ্রুত করে?**
একটি index হলো একটি আলাদা, সংগঠিত data structure (প্রায়ই sorted বা hashed) যা column value-কে row-এর physical অবস্থানে map করে, ঠিক যেমন একটি বইয়ের শেষে থাকা index। এটি database-কে পুরো table স্ক্যান করার পরিবর্তে সরাসরি মিলে যাওয়া row-এ চলে যেতে দেয়, একটি O(n) full scan-কে একটি O(log n) বা O(1) lookup-এ পরিণত করে।

**২. একটি index যোগ করার প্রধান খরচ কী? কেন আপনার প্রতিটি column index করা উচিত নয়?**
Index write amplification যোগ করে (প্রতিটি insert/update/delete-এ প্রতিটি index-কেও update করতে হয়), অতিরিক্ত storage, এবং চলমান maintenance (rebalancing, মাঝে মাঝে rebuild)। প্রতিটি column index করলে সব write ধীর হয়ে যাবে এবং storage ফুলে উঠবে এমন সুবিধার জন্য যা হয়তো কখনো ব্যবহৃতই হবে না, তাই index শুধুমাত্র সেসব column-এর জন্য যোগ করা উচিত যা প্রকৃতপক্ষে filter, join, বা sort-এ ব্যবহৃত হয়।

**৩. একটি B+Tree index-এর গঠন বর্ণনা করুন এবং ব্যাখ্যা করুন কেন এটি balanced।**
একটি B+Tree-তে একটি root node থাকে, internal node যা signpost হিসেবে কাজ করে, এবং leaf node যাতে row-এর pointer সহ প্রকৃত sorted value থাকে; leaf node-গুলো sorted order-এ একসাথে linked থাকে। এটি balanced কারণ root থেকে leaf পর্যন্ত প্রতিটি path একই দৈর্ঘ্যের, যা আপনি কোন value খুঁজছেন তা নির্বিশেষে ধারাবাহিক O(log n) performance নিশ্চিত করে।

**৪. কেন B-Tree range query-র জন্য ভালো কিন্তু hash index নয়?**
B+Tree leaf node sorted order-এ সংরক্ষিত এবং একসাথে linked থাকে, তাই একবার আপনি একটি range-এর শুরু খুঁজে পেলে, আপনি leaf-গুলোর মধ্য দিয়ে পাশাপাশি হেঁটে সব মিলে যাওয়া value সংগ্রহ করতে পারেন। Hash index একটি hash function-এর মাধ্যমে value-কে bucket-এ map করে, যা ইচ্ছাকৃতভাবে order নষ্ট করে দেয় — সংখ্যাগতভাবে বা বর্ণানুক্রমে কাছাকাছি থাকা দুটি value সম্পূর্ণ ভিন্ন bucket-এ পড়তে পারে, তাই "X এবং Y-এর মধ্যে সবকিছু" দক্ষভাবে সংগ্রহ করার কোনো উপায় নেই।

**৫. একটি B-Tree বনাম একটি hash index-এ lookup-এর time complexity কী?**
একটি B-Tree lookup হলো O(log n), কারণ আপনি একটি নির্দিষ্ট সংখ্যক balanced tree level নিচে নামেন। একটি hash index lookup গড়ে O(1), কারণ hash function সরাসরি bucket-এর অবস্থান গণনা করে, যদিও ভারী collision এটিকে খারাপ করে দিতে পারে।

**৬. একটি বাস্তব system-এর উদাহরণ দিন যা hash index বা hash table ব্যবহার করে, এবং ব্যাখ্যা করুন কেন hashing সেখানে উপযুক্ত।**
Redis তার অনেক মূল data structure-এর জন্য অভ্যন্তরীণভাবে hash table ব্যবহার করে, এবং Python-এর `dict` বা Java-এর `HashMap`-এর মতো in-memory language construct hash table-এর ওপর নির্মিত। এগুলো উপযুক্ত কারণ প্রধান access pattern হলো pure key-based equality lookup যেখানে range query বা sorted iteration-এর কোনো প্রয়োজন নেই, তাই hashing-এর গড়ে O(1) lookup একটি স্পষ্ট সুবিধা।

**৭. একটি composite (multi-column) index কী, এবং কেন column order গুরুত্বপূর্ণ?**
একটি composite index একাধিক column জুড়ে তৈরি করা হয়, যেমন `(last_name, first_name)`। এটি দক্ষভাবে সেই query-গুলোর সেবা দেয় যা column-গুলোর একটি left-prefix-এর ওপর filter করে — শুধু `last_name`, বা `last_name AND first_name` একসাথে — কিন্তু শুধুমাত্র `first_name`-এর ওপর filter করা কোনো query-কে দক্ষভাবে সেবা দিতে পারে না, কারণ index শারীরিকভাবে প্রথমে `last_name` দিয়ে সাজানো।

**৮. একটি covering index কী, এবং এটি কোন সমস্যার সমাধান করে?**
একটি covering index একটি query-এর প্রয়োজনীয় প্রতিটি column অন্তর্ভুক্ত করে (`SELECT`, `WHERE`, এবং `ORDER BY`-তে), ফলে database table-এ দ্বিতীয় কোনো lookup ছাড়াই সম্পূর্ণভাবে index থেকে query-এর উত্তর দিতে পারে (একটি "index-only scan")। এটি hot, read-heavy query-র জন্য I/O এবং latency কমায়, খরচ হলো একটি বড় index।

**৯. কোন পরিস্থিতিতে আপনার একটি index যোগ করা এড়ানো উচিত, যদিও এটি technically একটি query-কে দ্রুত করবে?**
Write-heavy table-এ (যেমন, logging/event table), যেখানে অতিরিক্ত index প্রতিটি insert-কে ধীর করে দেয়; ছোট table-এ, যেখানে একটি full scan ইতিমধ্যেই যথেষ্ট দ্রুত যে index-এর overhead তার মূল্য পুষিয়ে দেয় না; এবং low-cardinality column-এ (যেমন, একটি boolean flag), যেখানে index খুব বেশি search সংকীর্ণ করতে পারে না কারণ অনেক row একই value শেয়ার করে।

**১০. একটি table-এ ৫০ কোটি row আছে, এবং `email`-এর ওপর filter করা query ধীর। আপনি কী করবেন, এবং trade-off কী?**
প্রথমে `EXPLAIN`/query plan analysis দিয়ে নিশ্চিত করুন যে query `email`-এ missing index-এর কারণে একটি full table scan করছে। `email`-এর ওপর একটি B-Tree index যোগ করুন (অথবা query যদি অন্য একটি column-এর ওপর filter/sort-ও করে তাহলে একটি composite index), কারণ email lookup সাধারণত exact-match বা prefix-based এবং ordering থেকে উপকৃত হয়। Trade-off: `email`-কে স্পর্শ করা প্রতিটি insert/update এখন index-কেও write করে, storage বৃদ্ধি পায়, এবং table যদি অত্যন্ত write-heavy হয় তাহলে যোগ করা index-এর খরচকে read-latency-এর সুবিধার বিপরীতে বিবেচনা করতে হবে — কিন্তু এই ধরনের lookup-style query-র জন্য, read-এর গতি বৃদ্ধি (সেকেন্ড থেকে সম্ভাব্যভাবে মিলিসেকেন্ডে) সাধারণত write overhead-কে অনেক বেশি ছাড়িয়ে যায়।

**১১. filtered column-এ একটি index থাকা সত্ত্বেও database কেন সেটি ব্যবহার না করার সিদ্ধান্ত নিতে পারে?**
Query planner statistics-এর ভিত্তিতে খরচ অনুমান করে; যদি filter table-এর একটি বড় অংশের সাথে মিলে যায় (low selectivity), তাহলে লক্ষ লক্ষ row-এর জন্য index এবং table-এর মধ্যে লাফানোর চেয়ে একটি sequential scan প্রকৃতপক্ষে সস্তা হতে পারে (এই "random I/O" খরচই কারণ কেন planner মাঝে মাঝে low-cardinality বা কম selective column-এর index উপেক্ষা করে)। Stale statistics বা একটি অমিলযুক্ত query pattern (যেমন, একটি composite index-এর non-leading column-এ filter করা) planner-কে index এড়িয়ে যেতেও বাধ্য করতে পারে।

**১২. ধারণাগতভাবে, একটি clustered index এবং একটি regular (secondary) index-এর মধ্যে পার্থক্য কী?**
একটি clustered index table-এর row-গুলোর নিজেদের physical storage order নির্ধারণ করে (table data index structure-এর ভেতরেই থাকে, সাধারণত primary key-তে) — একটি table-এ মাত্র একটিই থাকতে পারে। একটি secondary (non-clustered) index হলো একটি আলাদা structure যা indexed value-এর পাশাপাশি প্রকৃত row-এর অবস্থানের pointer সংরক্ষণ করে, এবং বিভিন্ন query pattern-কে দ্রুত করতে একটি table-এ এমন অনেকগুলো থাকতে পারে।
