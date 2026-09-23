# Transaction Isolation Levels & Concurrency Control: Locking vs. MVCC

**Difficulty:** Advanced

## Learning Objectives

- একই ডেটার ওপর দুটি transaction যখন একসাথে কাজ করে, তখন database লেভেলে race condition দেখতে কেমন হয় তা ব্যাখ্যা করা।
- চারটি standard SQL isolation level এবং প্রতিটি কোন কোন anomaly প্রতিরোধ করে তা বর্ণনা করা।
- pessimistic locking (আগে lock নাও, তারপর কাজ করো) এবং optimistic concurrency control (আগে কাজ করো, তারপর check করো)-এর তুলনা করা।
- MVCC (Multi-Version Concurrency Control) কী এবং কেন এটি reader-দের writer-দের block না করে চলতে দেয়, তা ব্যাখ্যা করা।
- একটি নির্দিষ্ট system design পরিস্থিতির জন্য উপযুক্ত isolation level এবং concurrency strategy বেছে নেওয়া।

## Script

### Hook / Intro

Module 3-এ আমরা ACID-এর "I"-কে "Isolation" হিসেবে সংজ্ঞায়িত করেছিলাম — একসাথে চলা transaction-গুলো একে অপরের সাথে হস্তক্ষেপ করবে না — এবং এগিয়ে গিয়েছিলাম। সেই সংজ্ঞাটি সত্যি, কিন্তু বাস্তব interview বা বাস্তব production incident-এ এটি প্রায় কোনো কাজেই আসে না। ঠিক একই মিলিসেকেন্ডে দুজন গ্রাহক যখন একটি ফ্লাইটের শেষ সিটটি বুক করার চেষ্টা করে, তখন "হস্তক্ষেপ করবে না" কথাটার আসল মানে কী? আজ আমরা isolation-কে সত্যিকার অর্থে খুলে দেখব: transaction-গুলো একে অপরের ওপর ওভারল্যাপ করলে কোন কোন নির্দিষ্ট anomaly ঘটতে পারে, কোন isolation level performance-এর বিনিময়ে correctness দেয়, এবং সম্পূর্ণ ভিন্ন দুটি engineering পদ্ধতি — locking এবং MVCC — যেগুলো দিয়ে database আপনার বেছে নেওয়া যেকোনো level কার্যকর করে।

### The Core Problem: Concurrent Transactions

একটি transaction হলো একগুচ্ছ operation যা একটি একক atomic unit হিসেবে আচরণ করা উচিত। single-user জগতে transaction-গুলো কখনো ওভারল্যাপ করে না, তাই এখানে চিন্তা করার কিছু নেই। কিন্তু যখনই আপনার concurrent user থাকে — যা প্রতিটি বাস্তব সিস্টেমেই থাকে — তখন দুটি transaction একই সাথে overlapping ডেটা read এবং write করতে পারে, এবং কোনো protection না থাকলে বেশ কিছু নির্দিষ্ট anomaly ঘটতে পারে।

**Dirty read**: Transaction A এমন একটি row read করে যা transaction B পরিবর্তন করেছে কিন্তু এখনও commit করেনি। এরপর B যদি rollback করে, তাহলে A এমন ডেটার ওপর কাজ করে ফেলেছে যা আসলে কখনো ছিলই না। **Non-repeatable read**: Transaction A একই row দুইবার read করে এবং প্রতিবার আলাদা value পায়, কারণ A-এর দুটি read-এর মাঝখানে B সেই row-তে পরিবর্তন commit করেছে। **Phantom read**: Transaction A একই query দুইবার চালায় (যেমন, "$100-এর বেশি সব order গণনা করো"), এবং দ্বিতীয়বার ভিন্ন সেট row পায়, কারণ মাঝখানে B সেই শর্তের সাথে মিলে যাওয়া একটি row insert বা delete করেছে। এদের প্রতিটি হলো concurrency যেভাবে এমন ফলাফল তৈরি করতে পারে যা transaction-গুলো একে একে চললে কখনোই ঘটত না, তার একটি নির্দিষ্ট, স্পষ্টভাবে সংজ্ঞায়িত উপায়।

### The Four SQL Isolation Levels

SQL স্ট্যান্ডার্ড চারটি isolation level সংজ্ঞায়িত করে, প্রতিটি আগেরটির তুলনায় কম anomaly অনুমতি দেয়, বিনিময়ে বেশি coordination overhead-এর মূল্যে:

**Read Uncommitted** — সবচেয়ে দুর্বল level। Transaction-গুলো একে অপরের uncommitted পরিবর্তন দেখতে পারে। Dirty read সম্ভব। বাস্তবে খুব কমই ব্যবহৃত হয়; কিছু database (যেমন PostgreSQL) এটিকে Read Committed থেকে আলাদাভাবে implement-ই করে না।

**Read Committed** — বেশিরভাগ production database-এর (PostgreSQL, SQL Server, Oracle) default। একটি transaction শুধুমাত্র সেই ডেটা দেখতে পায় যা অন্য transaction-গুলো ইতিমধ্যে commit করেছে। Dirty read প্রতিরোধ করা হয়, কিন্তু non-repeatable read এবং phantom read তখনও ঘটতে পারে, কারণ transaction-এর ভেতরের প্রতিটি read সেই মুহূর্তের সর্বশেষ committed অবস্থা দেখে, যা একটি read থেকে আরেকটি read-এর মাঝে বদলে যেতে পারে।

**Repeatable Read** — MySQL-এর (InnoDB-এর মাধ্যমে) default। একবার একটি transaction কোনো row read করলে, transaction-এর বাকি সময় জুড়ে সেই row-এর জন্য একই value দেখার নিশ্চয়তা থাকে, এমনকি অন্য কোনো transaction সেটিতে পরিবর্তন commit করলেও। এটি non-repeatable read প্রতিরোধ করে। SQL স্ট্যান্ডার্ডের সংজ্ঞা অনুযায়ী phantom read তখনও প্রযুক্তিগতভাবে সম্ভব, যদিও বেশ কিছু database-এর প্রকৃত implementation (যেমন InnoDB-এর) যেকোনোভাবে বেশিরভাগ phantom পরিস্থিতি প্রতিরোধ করে।

**Serializable** — সবচেয়ে শক্তিশালী level। Transaction-গুলো প্রকৃতপক্ষে একসাথে চললেও, ঠিক এমনভাবে আচরণ করে যেন সেগুলো কোনো এক serial order-এ একে একে চলছে। তিনটি anomaly-ই প্রতিরোধ করা হয়। এর খরচ সবচেয়ে বেশি: এটি নিশ্চিত করতে database-কে সবচেয়ে বেশি কাজ করতে হয় — ব্যাপক locking অথবা আগ্রাসী conflict detection — যা throughput কমায় এবং transaction abort হয়ে retry করার সম্ভাবনা বাড়ায়।

চারটি level জুড়ে একই ধরনের trade-off দেখা যায় যা আমরা এই কোর্স জুড়ে দেখেছি: শক্তিশালী correctness guarantee মানেই বেশি coordination এবং throughput-এর খরচ। isolation level বেছে নেওয়া একটি স্পষ্ট trade-off সিদ্ধান্ত, এমন কোনো default নয় যা না পরীক্ষা করেই রেখে দেওয়া উচিত।

### Pessimistic Locking vs. Optimistic Concurrency Control

isolation প্রকৃতপক্ষে কার্যকর করার জন্য মূলত সম্পূর্ণ ভিন্ন দুটি engineering কৌশল আছে।

**Pessimistic locking** ধরে নেয় যে conflict হওয়ার সম্ভাবনা বেশি, তাই এটি আগে থেকেই সেগুলো প্রতিরোধ করে: একটি transaction কোনো row স্পর্শ করার আগেই সেটির ওপর একটি lock নেয়, এবং একই row স্পর্শ করতে চাওয়া অন্য যেকোনো transaction-কে lock release না হওয়া পর্যন্ত অপেক্ষা করতে হয়। এটি চিন্তা করা সহজ এবং গঠনগতভাবে নিরাপদ, কিন্তু এর খরচ throughput-এ পড়ে — transaction-গুলো lock-এর জন্য অপেক্ষায় লাইন দেয়, এবং lock ধরে থাকা কোনো transaction ধীর হলে (বা আরও খারাপ, দুটি transaction একে অপরের প্রয়োজনীয় lock নিজেরা ধরে থাকলে), তখন contention বা এমনকি **deadlock** তৈরি হয়, যা database-কে detect করে এবং একটি transaction abort করে সমাধান করতে হয়।

**Optimistic concurrency control (OCC)** ধরে নেয় যে conflict বিরল, তাই এটি কোনো কিছু lock না করেই transaction-গুলোকে এগিয়ে যেতে দেয়, এবং commit করার ঠিক আগে conflict চেক করে — সাধারণত row-এর একটি version number বা timestamp-কে transaction শুরু হওয়ার সময়কার মানের সাথে তুলনা করে। কিছু না বদলালে commit সফল হয়। কিছু বদলে গেলে transaction abort হয় এবং application সেটি retry করে। conflict সত্যিই বিরল হলে এটি locking-এর throughput খরচ এড়ায়, কিন্তু যখনই একটি conflict ঘটে তখন কাজ নষ্ট হয় (এবং application-এ retry logic প্রয়োজন হয়)। সঠিক পছন্দ পুরোপুরি আপনার প্রকৃত contention প্যাটার্নের ওপর নির্ভর করে: একটি high-contention hot row (যেমন একটি জনপ্রিয় পণ্যের inventory count) pessimistic locking-এর পক্ষে যায়; একটি low-contention পরিস্থিতি (বেশিরভাগ row, বেশিরভাগ সময়, বেশিরভাগ application-এ) optimistic concurrency-র পক্ষে যায়, এবং এই কারণেই অনেক ORM-এর "optimistic locking" ফিচারে এটিই default ধারণা।

### MVCC: How Modern Databases Actually Do This

বেশিরভাগ production relational database (PostgreSQL, MySQL/InnoDB, Oracle) শুধুমাত্র reader-দের বাইরে রাখার জন্য locking-এর ওপর নির্ভর করে না — তারা **Multi-Version Concurrency Control (MVCC)** ব্যবহার করে। একটি row-এর একক বর্তমান value থাকা এবং reader-writer-রা সেটার জন্য লড়াই করার বদলে, database একটি row-এর একাধিক version রাখে, প্রতিটিকে সেই transaction দিয়ে ট্যাগ করা হয় যেটি সেটি তৈরি করেছে। একটি transaction যখন কোনো row read করে, এটি "বর্তমান value" দেখে না — এটি সেই version দেখে যা তার নিজের transaction (বা statement) শুরু হওয়ার সময় commit করা ছিল। এর মানে হলো একটি দীর্ঘস্থায়ী read কখনো একটি concurrent write-কে block করার প্রয়োজন হয় না, এবং একটি write কখনো একটি concurrent read-কে block করার প্রয়োজন হয় না, কারণ তারা আক্ষরিক অর্থেই একই logical row-এর ভিন্ন version দেখছে। Writer-দের এখনও অন্য writer-দের সাথে coordinate করতে হয় (একই row-এর একটি conflicting update-এ দুটি transaction একসাথে "জয়ী" হতে পারে না), কিন্তু pure locking যে classic reader-vs-writer contention তৈরি করে, তা প্রায় সম্পূর্ণ উধাও হয়ে যায়। খরচ হলো: পুরোনো row version-গুলো, যখন আর কোনো transaction-এর প্রয়োজন থাকে না, তখন সেগুলো পরিষ্কার করতে হয় — এটিই ঠিক PostgreSQL-এর `VACUUM` প্রসেস করে, এবং এই কারণেই নিয়মিত vacuuming ছাড়া বিপুল সংখ্যক update তৈরি করা কোনো application table bloat-এ ভুগতে পারে।

### Real-World Example

আমাদের ফ্লাইট-বুকিং পরিস্থিতিটি বিবেচনা করি: দুজন গ্রাহক ঠিক একই মুহূর্তে শেষ সিটটি বুক করার চেষ্টা করছে। Pessimistic locking-সহ Read Committed-এর অধীনে, seat row-তে প্রথমে পৌঁছানো transaction সেটি lock করে, available-seats কাউন্টার কমায় এবং commit করে; দ্বিতীয় transaction, lock-এর জন্য অপেক্ষা করার পর, এখন শূন্য সিট available দেখে এবং পরিষ্কারভাবে fail করে — কোনো double-booking হয় না। একটি MVCC-ভিত্তিক Repeatable Read পদ্ধতির অধীনে, উভয় transaction হয়তো তাদের নিজস্ব consistent snapshot থেকে "1 সিট available" পড়ে শুরু করতে পারে, কিন্তু underlying write path-এর তখনও দ্বিতীয় transaction-এর প্রকৃত update-এর দরকার হয় এটা detect করার জন্য যে row-টি সেটি read করার পর থেকে বদলে গেছে, যা একটি serialization failure ঘটায় যা application-কে catch করে retry করতে হয় — এই কারণেই high-stakes আর্থিক ও inventory সিস্টেমগুলো সাধারণত শুধু MVCC-এর isolation-কে বিশ্বাস করে চলে যেতে পারে না; correctness যেখানে সত্যিই কোনো race সহ্য করতে পারে না, ঠিক সেই নির্দিষ্ট পয়েন্টগুলোতে তাদের এখনও স্পষ্ট conflict-handling logic প্রয়োজন হয় (উদাহরণস্বরূপ, ঠিক এই row-এর ওপর pessimistic locking জোর করার জন্য একটি `SELECT ... FOR UPDATE`)।

### Recap

সুরক্ষা ছাড়া রাখা হলে concurrent transaction-গুলো dirty read, non-repeatable read এবং phantom read তৈরি করতে পারে। চারটি SQL isolation level — Read Uncommitted, Read Committed, Repeatable Read, Serializable — এদের মধ্যে কোনগুলো প্রতিরোধ করা হবে তা coordination overhead এবং throughput-এর বিনিময়ে trade-off করে। Pessimistic locking contention এবং সম্ভাব্য deadlock-এর খরচে আগে থেকেই conflict প্রতিরোধ করে; optimistic concurrency control transaction-গুলোকে এগিয়ে যেতে দেয় এবং শুধু commit-এর সময় conflict চেক করে, low-contention workload-এর পক্ষে ভালো। MVCC-ই হলো যেভাবে বেশিরভাগ আধুনিক database বাস্তবে isolation implement করে — একাধিক row version রেখে যাতে reader ও writer একে অপরকে block না করে — কিন্তু hot, high-stakes row-গুলোর তখনও প্রায়শই এর ওপরে স্পষ্ট locking প্রয়োজন হয়।

### What's Next

আমরা গভীরভাবে দেখেছি কীভাবে একটি database concurrent transaction-গুলোর মধ্যে correctness নিশ্চিত করে। পরের video-তে, আমরা আরও এক স্তর নিচে যাব — একটি database-এর storage engine ডেটা ডিস্কে সংরক্ষণ ও index করার জন্য যে প্রকৃত data structure ব্যবহার করে, এবং কেন write-heavy সিস্টেমগুলো Module 3-এ পরিচয় করানো B-tree-র থেকে একেবারে ভিন্ন একটি structure বেছে নেয়, সেদিকে।

## Key Takeaways

- সুরক্ষাহীন concurrent transaction-গুলো dirty read, non-repeatable read এবং phantom read তৈরি করতে পারে — প্রতিটিই একটি নির্দিষ্ট, স্পষ্টভাবে সংজ্ঞায়িত anomaly।
- চারটি SQL isolation level (Read Uncommitted, Read Committed, Repeatable Read, Serializable) ধীরে ধীরে বেশি anomaly প্রতিরোধ করে, বিনিময়ে ধীরে ধীরে বেশি coordination খরচে।
- Pessimistic locking আগে থেকেই conflict প্রতিরোধ করে (নিরাপদ, কিন্তু throughput-এর খরচ ও deadlock-এর ঝুঁকি আছে); optimistic concurrency control শুধু commit-এর সময় conflict চেক করে (low contention-এ দ্রুত, retry logic প্রয়োজন)।
- MVCC প্রতিটি row-এর একাধিক version রেখে বেশিরভাগ আধুনিক database-কে classic reader-vs-writer blocking এড়াতে দেয়, বিনিময়ে background cleanup প্রয়োজন হয় (যেমন, PostgreSQL-এর `VACUUM`)।
- Isolation level এবং concurrency strategy হলো স্পষ্ট engineering trade-off — high-contention, high-stakes row-গুলোর (যেমন inventory count) MVCC database-এও প্রায়ই স্পষ্ট locking প্রয়োজন হয়।
