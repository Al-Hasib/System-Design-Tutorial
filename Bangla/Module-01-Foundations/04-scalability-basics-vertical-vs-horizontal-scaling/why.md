# Why This Topic Matters: Scalability Basics — Vertical vs Horizontal Scaling

> **এক বাক্যে:** বৃদ্ধি (growth) হলো সাফল্যের স্বাভাবিক পরিণতি, এবং একটি সিস্টেম যা শুধুমাত্র একটি বড় machine কিনেই বাড়তে পারে, তার একটি কঠিন সীমা (hard ceiling), একটি single point of failure, এবং একটি price curve থাকে যা শেষ পর্যন্ত খাড়াভাবে (vertical) ওপরে ওঠে।

## The World Before This Idea

"server struggle করছে" শুনে স্বাভাবিক প্রতিক্রিয়া হলো "একটি বড় server নাও।" এটি কাজ করে — কিছুদিনের জন্য। এটি সবচেয়ে দ্রুততম সম্ভাব্য সমাধান, এতে কোনো code পরিবর্তনের প্রয়োজন হয় না, এবং architecture ব্লগগুলো যতটা স্বীকার করে তার চেয়ে বেশি ক্ষেত্রেই এটি প্রকৃতপক্ষে সঠিক সিদ্ধান্ত।

তারপর এই ঘটনাগুলোর যেকোনো একটি ঘটে: আপনার cloud provider যে বৃহত্তম instance বিক্রি করে তা আর যথেষ্ট নয়; পরের size-টি 1.3x performance-এর জন্য 2x খরচ করে; অথবা একক বড় machine-টি reboot হয় এবং আপনার পুরো product offline হয়ে যায়। সেই মুহূর্তে "আরও বড় একটি কিনুন" আর একটি strategy থাকে না, এবং architecture-কে পরিবর্তন করতে হয় — সাধারণত চাপের মধ্যে, সাধারণত খারাপভাবে।

## The Problems It Solves

### ১. কঠিন সীমা (The hard ceiling)
**আপনি যা দেখেন:** আপনি ইতিমধ্যে সর্ববৃহৎ উপলব্ধ instance type-এ আছেন। Traffic এখনো বাড়ছে। পরবর্তী কোনো ধাপ নেই।

**কেন এটি ঘটে:** Vertical scaling পদার্থবিজ্ঞান (physics) এবং vendor-রা প্রকৃতপক্ষে যা তৈরি করে তার দ্বারা সীমাবদ্ধ। আপনি একটি অসীম বড় machine কিনতে পারবেন না, এবং range-এর শীর্ষে একটি খাড়া premium দাম থাকে।

**Horizontal scaling কীভাবে এটি সমাধান করে:** একটি machine 100 unit কাজ করার বদলে, দশটি machine প্রতিটি 10 unit করে করে। Machine-এর সংখ্যার উপর কোনো সীমা নেই, এবং প্রতিটি একটি সস্তা commodity instance। Capacity বড় বড় জিনিস খোঁজার বদলে box যোগ করার বিষয়ে পরিণত হয়।

### ২. Single point of failure
**আপনি যা দেখেন:** একটি kernel patch-এর জন্য একটি server reboot হয় এবং product ছয় মিনিটের জন্য বন্ধ থাকে। একটি disk ব্যর্থ হয় এবং এটি ঘণ্টার পর ঘণ্টা বন্ধ থাকে।

**কেন এটি ঘটে:** Vertical scaling সবকিছু একটি machine-এ কেন্দ্রীভূত করে। সেই machine-কে বড় করা outage-কে কম নয়, *আরও* ব্যয়বহুল করে তোলে — আপনি একই ঝুড়িতে আরও বেশি ডিম রেখেছেন।

**Horizontal scaling কীভাবে এটি সমাধান করে:** একটি load balancer-এর পেছনে দশটি অভিন্ন node থাকলে, একটি মারা গেলে service-এর 100% নয়, capacity-র 10% সরে যায়। Redundancy scaling strategy-র একটি পার্শ্ব-প্রতিক্রিয়া (side effect) হয়ে ওঠে, এবং deploy-গুলো একসাথে না করে rolling হতে পারে।

### ৩. দিনে চব্বিশ ঘণ্টা peak-এর জন্য টাকা দেওয়া
**আপনি যা দেখেন:** আপনার infrastructure bill Black Friday-র জন্য size করা, আর এখন ফেব্রুয়ারি মাস।

**কেন এটি ঘটে:** একটি vertically scale করা সিস্টেমকে অবশ্যই এর সবচেয়ে খারাপ ঘণ্টার জন্য স্থায়ীভাবে provision করতে হয়, কারণ একটি machine resize করার অর্থ downtime, এবং এটি মিনিটের মধ্যে ঘটতে পারে না।

**Horizontal scaling কীভাবে এটি সমাধান করে:** অভিন্ন node যোগ ও অপসারণ করা দ্রুত এবং বিঘ্নহীন, যা autoscaling সম্ভব করে তোলে — peak-এ বারোটি node, রাতভর তিনটি। আপনি প্রকৃতপক্ষে যে load আছে তার জন্যই টাকা দেন।

### ৪. সবচেয়ে খারাপ মুহূর্তে আবিষ্কার করা যে আপনার code scale out করতে পারে না
**আপনি যা দেখেন:** আপনি একটি দ্বিতীয় application server যোগ করেন এবং জিনিসগুলো ভেঙে পড়ে: session হারিয়ে যায়, আপলোড করা file অর্ধেক ব্যবহারকারীর জন্য অনুপস্থিত, একটি scheduled job হঠাৎ দুবার চলে।

**কেন এটি ঘটে:** একটি একক machine-এর জন্য লেখা code নীরবে local state জমা করে — in-memory session, local disk-এ file, in-process scheduler, local lock। এর কোনোটিই replicate করা টিকে থাকে না।

**এই topic-টি কীভাবে এটি সমাধান করে:** জানা যে horizontal scaling হলো আপনার গন্তব্য, statelessness-কে প্রথম দিন থেকেই একটি design নিয়ম বানিয়ে দেয়। এটি শুরুতে প্রায় কিছুই খরচ করে না এবং পরে retrofit করা কষ্টকর।

## The Price You Pay

Horizontal scaling বিনামূল্যে নয়, এবং vertical scaling সেকেলে নয়:

- **Distributed complexity।** এখন আপনার load balancing, service discovery, shared session storage, aggregated logging, এবং partial failure নিয়ে চিন্তা করার একটি উপায় প্রয়োজন। এই প্রতিটিই একটি নতুন component যা ভেঙে পড়তে পারে।
- **Data হলো কঠিন অংশ।** Stateless app server সহজে scale out করে; database করে না। Replication, sharding, এবং consistency trade-off — এই সবকিছু আপনি চেষ্টা করার মুহূর্তেই সামনে আসে।
- **Vertical scaling প্রায়ই কেবল সঠিক উত্তর।** এটি তাৎক্ষণিক, কোনো architectural পরিবর্তনের প্রয়োজন হয় না, এবং আধুনিক machine বিশাল। অনেক real system-এর জন্য, একটি বড়, ভালোভাবে tune করা database server একটি distributed cluster-এর চেয়ে সহজ এবং সস্তা — এবং সরলতার প্রকৃত, অনাড়ম্বর মূল্য আছে।

পরিণত অবস্থান: যতক্ষণ এটি এখনো সস্তা এবং নিরাপদ ততক্ষণ vertically scale করুন, কিন্তু *code এমনভাবে লিখুন যেন আপনি horizontally scale করবেন*, যাতে প্রয়োজনের সময় transition-টি উপলব্ধ থাকে।

## When You Need It — and When You Don't

| Horizontally scale করুন যখন | Vertically scale করুন যখন |
|---|---|
| আপনার redundancy এবং zero-downtime deploy প্রয়োজন | আপনার আজই সমাধান দরকার, কোনো code পরিবর্তন ছাড়া |
| Load অনিয়মিত (spiky) এবং autoscaling সত্যিকারের টাকা বাঁচাবে | Workload একটি single-node database |
| আপনি সর্ববৃহৎ ব্যবহারিক instance-এ পৌঁছে গেছেন | সিস্টেম ছোট এবং সরলতাই জেতে |
| Workload stateless এবং সহজে parallelize করা যায় | Workload প্রকৃতপক্ষে parallelize করা কঠিন |

## Why This Shows Up in Interviews

"আপনি এটি কীভাবে scale করবেন?" প্রায় প্রতিটি system design round-এর মেরুদণ্ড। একটি দুর্বল উত্তর বলে "আরও server যোগ করুন।" একটি শক্তিশালী উত্তর stateless tier-কে (সহজ — load balancer-এর পেছনে node যোগ করুন) stateful tier থেকে (কঠিন — read-এর জন্য replica, write-এর জন্য sharding, এবং প্রতিটির সাথে যুক্ত একটি consistency trade-off) আলাদা করে, এবং এটি আসলে কোন bottleneck সমাধান করছে তা নির্দিষ্ট করে বলে।

## How It Connects

এই topic-টি পুরো course-এর কব্জা (hinge)। Horizontal scaling হলো *কারণ* কেন আপনার **load balancing** প্রয়োজন (কিছু একটিকে traffic বিতরণ করতে হবে), **distributed caching** (per-node local cache-গুলো ভিন্ন হয়ে যায়), **replication এবং sharding** (data tier-কেও scale করতে হবে), **consistent hashing** (একটি node যোগ করলে সবকিছু পুনর্বিন্যাস হওয়া উচিত নয়), এবং **CAP theorem** (একাধিক node মানে network partition সম্ভব)। নিচের দিকের প্রায় সবকিছুই একাধিক machine চালানোর সিদ্ধান্তের একটি পরিণতি।

**পরবর্তী:** [Availability, Reliability, Redundancy & Fault Tolerance](../05-availability-reliability-and-fault-tolerance/why.md) — কারণ আরও machine মানে আরও জিনিস যা ব্যর্থ হতে পারে।
