# এই বিষয়টি কেন গুরুত্বপূর্ণ: Database Indexing

> **এক বাক্যে:** Indexing হলো বেশিরভাগ system-এর সবচেয়ে বেশি প্রভাবশালী performance fix — একটি এক-লাইনের পরিবর্তন যা ৩০ সেকেন্ডের একটি query-কে ৩ মিলিসেকেন্ডের একটি query-তে পরিণত করে — এবং এটাই কারণ যে বেশিরভাগ "আমাদের database scale করতে হবে" আলোচনা অকালপক্ব।

## এই ধারণার আগের জগৎ

একটি index ছাড়া, একটি row খুঁজে পাওয়া মানে প্রতিটি row পড়া। ১,০০০ row-তে এটা ঠিক আছে কিন্তু ৫ কোটি row-তে এটা বিপর্যয়কর।

এই ব্যর্থতার ধরনটি বিপজ্জনক কারণ এটি development-এর সময় অদৃশ্য থাকে। ২০০ row-বিশিষ্ট একটি seed dataset-এর বিপরীতে একটি query index আছে কি নেই তা নির্বিশেষে সাথে সাথে ফলাফল দেয়। ছয় মাস পরে production-এ, ২ কোটি row-এর বিপরীতে সেই একই query ৪০ সেকেন্ড সময় নেয় এবং database CPU-কে সম্পূর্ণভাবে পূর্ণ করে ফেলে — আর এটি ধীর হওয়ার কারণে, connection জমতে থাকে অপেক্ষায়, connection pool শেষ হয়ে যায়, এবং তখন *অসম্পর্কিত* query-গুলোও ব্যর্থ হতে শুরু করে। একটি column-এ একটি missing index পুরো application-কে ধসিয়ে দিতে পারে।

## এটি যেসব সমস্যার সমাধান করে

### ১. Linear scan যা আপনার সাফল্যের সাথে সাথে বেড়ে যায়
**আপনি যা দেখেন:** যেসব response time launch-এর সময় ঠিক ছিল, তা ক্রমান্বয়ে খারাপ হতে থাকে এবং তারপর হঠাৎ ভেঙে পড়ে। কোডে কিছুই পরিবর্তন হয়নি।

**কেন এটা ঘটে:** একটি full table scan হলো O(n)। Table বড় হওয়ার সাথে সাথে, query আনুপাতিকভাবে ধীর হয়, যতক্ষণ না এটি এমন একটি threshold অতিক্রম করে যেখানে এটি আর memory-তে ফিট হয় না বা timeout-এর মধ্যে সম্পূর্ণ হয় না।

**Indexing কীভাবে এটি সমাধান করে:** একটি B-tree lookup হলো O(log n)। ১০ লাখ থেকে ১০০ কোটি row-এ যাওয়া একটি B-tree-কে প্রায় ৩ level থেকে প্রায় ৫ level-এ নিয়ে যায় — data-তে ১০০০ গুণ বৃদ্ধির জন্য কাজে ২ গুণেরও কম বৃদ্ধি। এটাই পার্থক্য একটি system-এর মধ্যে যেটা বৃদ্ধি সহ্য করে এবং যেটা করে না।

### ২. একটি ধীর query পুরো database-কে বিষাক্ত করে ফেলা
**আপনি যা দেখেন:** Checkout ব্যর্থ হতে শুরু করে। তদন্তে দেখা যায় checkout ঠিক আছে — index ছাড়া একটি analytics query connection ধরে রাখছে এবং বাকি সবকিছুকে ক্ষুধার্ত করছে।

**কেন এটা ঘটে:** একটি database-এর নির্দিষ্ট সংখ্যক connection, memory, এবং I/O bandwidth থাকে। একটি বিশাল table-এর scan এই তিনটিই খরচ করে এবং বোনাস হিসেবে buffer pool থেকে বাকি সবার cached page-ও উচ্ছেদ করে দেয়।

**Indexing কীভাবে এটি সমাধান করে:** আপত্তিকর query-টিকে সস্তা করে দেওয়া resource contention দূর করে। এ কারণেই "একটি index যোগ করো" প্রায়ই এমন উপসর্গ ঠিক করে দেয় যা দেখতে একেবারেই query সমস্যার মতো নয়।

### ৩. Sorting এবং range query যা optimize করে সরানো যায় না
**আপনি যা দেখেন:** একটি বড় table-এ `ORDER BY created_at DESC LIMIT 20` ধীর, যদিও এটি মাত্র ২০টি row ফেরত দেয়।

**কেন এটা ঘটে:** একটি ordered index ছাড়া, database-কে কোন ২০টি row সবচেয়ে সাম্প্রতিক তা জানার আগে *সবকিছু* পড়তে এবং sort করতে হয়।

**Indexing কীভাবে এটি সমাধান করে:** একটি B+Tree linked leaf-সহ sorted order-এ value সংরক্ষণ করে, তাই database সরাসরি range-এর সঠিক প্রান্তে হেঁটে ২০টি entry পড়তে পারে। এটাই ঠিক কারণ B-tree হ্যাশ index-কে ছাড়িয়ে যায়, যদিও pure equality-এর জন্য hashing দ্রুততর: বাস্তব application-গুলো সবসময় sort এবং range-scan করে।

### ৪. "সবকিছুতে একটি index যোগ করো" write-কে ভেঙে ফেলা
**আপনি যা দেখেন:** একটি performance push সবখানে index যোগ করার পর, read latency উন্নত হয়েছে কিন্তু write throughput অর্ধেক কমে গেছে।

**কেন এটা ঘটে:** প্রতিটি insert, update, এবং delete-এ প্রতিটি index-কে update করতে হয়। আটটি index মানে একটি logical write নয়টি physical write-এ পরিণত হয়, সাথে page split এবং rebalancing।

**এই বিষয়টি কীভাবে এটি সমাধান করে:** এটি indexing-কে একটি *trade* হিসেবে পুনর্গঠন করে — write খরচ ও storage দিয়ে কেনা read গতি — তাই আপনি সচেতনভাবে index করেন: high-cardinality column যেগুলো প্রকৃতপক্ষে filter, join, বা sort করা হয়, আর কিছু নয়।

## যে মূল্য আপনাকে দিতে হয়

- **Write amplification।** উপরে আলোচনা করা হয়েছে, এবং এটিই প্রধান। Write-heavy table (event, log, metric) সর্বনিম্ন প্রয়োজনীয় index set বহন করা উচিত।
- **Storage।** Index হলো প্রকৃত data structure। ব্যাপকভাবে indexed একটি table row-এর চেয়ে index-এ বেশি space খরচ করতে পারে।
- **Maintenance।** Fragmentation, bloat, এবং stale statistics সময়ের সাথে সাথে index-এর কার্যকারিতা কমিয়ে দেয়। এবং একটি বিশাল live table-এ একটি index তৈরি করলে সেটি lock হয়ে যেতে পারে — এটি একটি operational ঝুঁকি যার জন্য online/concurrent index build প্রয়োজন।
- **যেসব index planner ব্যবহার করবে না।** `(a, b)`-এর ওপর একটি composite index শুধু `b`-এর ওপর filter করা query-র জন্য কিছুই করে না। একটি low-selectivity column-এর index-কে উপেক্ষা করা হয় কারণ একটি scan প্রকৃতপক্ষে সস্তা। এমন index তৈরি করা যা আপনি মনে করেন সাহায্য করছে অথচ শুধু আপনার খরচই বাড়াচ্ছে, এটা প্রচলিত।

## কখন এটি প্রয়োজন — এবং কখন নয়

| Index করুন যখন | Index করবেন না যখন |
|---|---|
| Column `WHERE`, `JOIN`, বা `ORDER BY`-তে দেখা যায় | Column কখনো filter বা sort করা হয় না |
| Cardinality বেশি (email, user ID, timestamp) | Cardinality কম (boolean, তিন-মান বিশিষ্ট status) |
| Table বড় এবং read-heavy | Table খুবই ছোট — একটি scan ইতিমধ্যেই দ্রুত |
| একটি query plan নিশ্চিত করে যে sequential scan হলো bottleneck | Table write-প্রধান এবং read কালেভদ্রে হয় |

## Interview-এ এটি কেন আসে

"এই query ধীর — আপনি কী করবেন?" একটি প্রচলিত প্রশ্ন, এবং প্রত্যাশিত প্রক্রিয়া হলো: query plan দেখুন, scan চিহ্নিত করুন, সঠিক index যোগ করুন, যাচাই করুন। গভীরতর follow-up প্রশ্নগুলো যাচাই করে আপনি *structure* বোঝেন কিনা: কেন B-tree range সমর্থন করে আর hash index করে না, কেন composite index-এ column order গুরুত্বপূর্ণ, একটি covering index কী দেয়, এবং write path-এ indexing-এর খরচ কী। যেসব candidate না জিজ্ঞাসা করেও write trade-off উল্লেখ করেন তারা আলাদা হয়ে ওঠেন, কারণ এটি দেখায় যে তারা শুধু database query করেননি, বরং সেটি পরিচালনাও করেছেন।

## এটি কীভাবে সংযুক্ত

Indexing হলো প্রথম যা চেষ্টা করা উচিত ভারী machinery-র দ্বারস্থ হওয়ার আগে — বেশিরভাগ system যাদের "sharding দরকার" মনে হয়, তাদের আসলে একটি index দরকার। এটি সরাসরি **SQL vs NoSQL** পছন্দের ওপর ভিত্তি করে তৈরি (NoSQL system-এর নিজস্ব index model এবং সীমাবদ্ধতা আছে), **caching** সিদ্ধান্তের ভিত্তি তৈরি করে (একটি indexed query হয়তো এতটাই দ্রুত হতে পারে যে invalidation-এর জটিলতার তুলনায় cache মূল্যবান নাও হতে পারে), এবং **LSM trees vs B-trees**-এর সাথে সংযুক্ত, যা এর নিচে থাকা storage-engine স্তরটি ব্যাখ্যা করে।

**পরবর্তী:** [Database Replication](../13-database-replication/why.md) — যখন একটি database node যথেষ্ট নয়, বা যথেষ্ট নিরাপদ নয়, তখন কী করতে হবে।
