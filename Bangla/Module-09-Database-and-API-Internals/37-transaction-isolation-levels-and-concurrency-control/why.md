# Why This Topic Matters: Transaction Isolation Levels & Concurrency Control

> **এক বাক্যে:** আপনার database-এর একটি default isolation level আছে যা আপনি প্রায় নিশ্চিতভাবেই নিজে বেছে নেননি, এবং এটি নির্দিষ্ট কিছু anomaly-র অনুমতি দেয় যা শেষ পর্যন্ত এমন একটি bug তৈরি করবে যা আপনি reproduce করতে পারবেন না — কারণ এটি শুধু তখনই ঘটে যখন দুজন user একই মুহূর্তে কাজ করে।

## The World Before This Idea

সবচেয়ে সাধারণ ঘটনা। কোড একটি balance read করে, সেটি যথেষ্ট কিনা check করে, বিয়োগ করে, এবং ফিরিয়ে লিখে:

```
balance = SELECT balance FROM accounts WHERE id = 1   -- reads 100
if balance >= 100: UPDATE accounts SET balance = 0 WHERE id = 1
```

100 balance-এর বিপরীতে 100-এর দুটি withdrawal একই সাথে চলে। দুটোই 100 read করে। দুটোই check পাস করে। দুটোই 0 লিখে। অ্যাকাউন্টটি 200 পরিশোধ করেছে। এটি একটি **lost update**, এবং কোডটি review-তে সম্পূর্ণ সঠিক দেখায়।

এবার কঠিন অংশ: এটি ঘটতে পারে কিনা তা নির্ভর করে isolation level-এর ওপর। Read Committed-এর অধীনে — PostgreSQL, Oracle, এবং SQL Server-এর default — এটি *ঘটতে পারে*। PostgreSQL-এ Repeatable Read-এর অধীনে, দ্বিতীয় transaction এর বদলে একটি serialization error পায়। একই কোড, ভিন্ন ফলাফল, একটি সেটিং দ্বারা নির্ধারিত যা বেশিরভাগ team কখনো দেখেই না।

## The Problems It Solves

### ১. এমন anomaly যা আপনি reproduce করতে পারবেন না
**আপনি যা দেখেন:** ডুপ্লিকেট বুকিং, নেগেটিভ inventory, ডাবল-স্পেন্ট credit, এমন একটি report যার সংখ্যাগুলো মিলছে না। এটি production-এ সপ্তাহে একবার ঘটে এবং testing-এ কখনোই ঘটে না।

**কেন এটি ঘটে:** এগুলো timing-নির্ভর। এগুলোর জন্য দুটি transaction-কে একটি নির্দিষ্ট পদ্ধতিতে interleave করতে হয়, যা কম concurrency-তে বিরল এবং বেশি concurrency-তে ধ্রুবক। Test suite সিরিয়ালি চলে এবং এগুলো কখনো তৈরি করে না।

**Isolation level কীভাবে এটি সমাধান করে:** এগুলো এই anomaly-গুলোকে নাম এবং একটি সংজ্ঞায়িত সীমা দেয়। **Dirty read** (uncommitted ডেটা দেখা), **non-repeatable read** (আপনার transaction-এর মধ্যেই একই row বদলে যাওয়া), **phantom read** (আপনার query-র সাথে মিলে যাওয়া নতুন row হাজির হওয়া), এবং **lost update** — প্রতিটির জন্যই একটি level আছে যেখানে সেটি প্রতিরোধ করা হয়। আপনার level কোন কোন anomaly-র অনুমতি দেয় তা জানা থাকলে, "এই bug ঘটতে পারে কি?" প্রশ্নটির উত্তর আপনি documentation থেকে দিতে পারবেন, production থেকে নয়।

### ২. এমন check-then-act race যা সঠিক দেখায়
**আপনি যা দেখেন:** এমন কোড যা read করে, সিদ্ধান্ত নেয়, এবং write করে — programming-এর সবচেয়ে স্বাভাবিক প্যাটার্ন — load-এর অধীনে ভুল ফলাফল তৈরি করছে।

**কেন এটি ঘটে:** read এবং write-এর মাঝের ফাঁকটাই সেই জায়গা যেখানে আরেকটি transaction ঢুকে পড়ে।

**Concurrency control কীভাবে এটি সমাধান করে:** হয় read করার সময় একটি lock নিন (`SELECT ... FOR UPDATE` — pessimistic), অথবা write করার সময় একটি version number-এর মাধ্যমে conflict detect করুন (`UPDATE ... WHERE version = 7` — optimistic), অথবা সিদ্ধান্তটিকে database-এর মধ্যে একটি atomic operation হিসেবে ঠেলে দিন (`UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100`)। তৃতীয় অপশনটি যে বিদ্যমান, তা জানা এবং সেটিকে প্রাধান্য দেওয়া, কোনো locking ছাড়াই পুরো একটি শ্রেণীর race দূর করে দেয়।

### ৩. Reader এবং writer একে অপরকে block করা
**আপনি যা দেখেন:** একটি দীর্ঘ analytical query shared lock ধরে রাখে এবং তার পেছনের প্রতিটি write আটকে যায়; অথবা একটি write exclusive lock ধরে রাখে এবং প্রতিটি reader অপেক্ষা করে।

**কেন এটি ঘটে:** pure lock-ভিত্তিক concurrency control reader এবং writer-কে পারস্পরিকভাবে exclusive করে তোলে।

**MVCC কীভাবে এটি সমাধান করে:** Multi-Version Concurrency Control প্রতিটি row-এর একাধিক version রাখে। একটি reader কোনো lock না নিয়েই তার transaction শুরু হওয়ার মুহূর্তের একটি consistent snapshot দেখে; একটি writer reader-দের block না করেই একটি নতুন version তৈরি করে। **Reader কখনো writer-কে block করে না, এবং writer কখনো reader-কে block করে না।** এই কারণেই PostgreSQL, MySQL InnoDB, এবং Oracle সবাই MVCC ব্যবহার করে, এবং এটি database engineering-এর সবচেয়ে উচ্চ-প্রভাবশালী ধারণাগুলোর একটি।

### ৪. Throughput-এর খরচে কেনা correctness
**আপনি যা দেখেন:** নিরাপদ থাকার জন্য সবকিছু Serializable-এ পাল্টে ফেলা, এবং transaction retry ও lock contention-এর অধীনে throughput ধসে পড়তে দেখা।

**কেন এটি ঘটে:** Serializable হলো সবচেয়ে শক্তিশালী guarantee এবং সবচেয়ে ব্যয়বহুল — হয় ভারী locking, অথবা ঘন ঘন serialization failure যা retry করতে হয়।

**এই topic কীভাবে এটি সমাধান করে:** এটি আপনাকে guarantee-টির পরিসর নির্ধারণ করতে দেয়। বেশিরভাগ application-কে default level-এ চালান, এবং শুধুমাত্র সেই নির্দিষ্ট transaction-গুলোর জন্য isolation বাড়ান (অথবা স্পষ্ট lock নিন) যেখানে একটি anomaly ব্যয়বহুল হতে পারে। প্রতিটি operation-এর জন্য correctness, পুরো সিস্টেমের জন্য নয় — ঠিক একই নীতি consistency model-এ দেখা যায়।

## The Price You Pay

- **উচ্চতর isolation-এর খরচ concurrency-তে পড়ে।** বেশি locking, বেশি blocking, retry করার জন্য বেশি aborted transaction। বিনামূল্যে correctness বলে কিছু নেই।
- **Optimistic concurrency যেখানেই ব্যবহৃত হয় সেখানেই retry logic প্রয়োজন।** উচ্চ contention-এর অধীনে, retry প্রাধান্য পেতে পারে এবং throughput pessimistic locking-এর চেয়ে খারাপ হতে পারে।
- **Pessimistic locking deadlock-এর ঝুঁকি নেয়।** দুটি transaction ভিন্ন ক্রমে একই lock নিলে deadlock হবে; database এটি detect করে একটিকে হত্যা করে, যা আপনার application-কে সামলাতে হয়। সামঞ্জস্যপূর্ণ lock ordering হলো স্ট্যান্ডার্ড প্রতিরোধ, এবং এটি একটি শৃঙ্খলা যা পুরো codebase জুড়ে বজায় রাখতে হয়।
- **MVCC আবর্জনা তৈরি করে।** পুরোনো row version জমা হয় এবং সেগুলো পরিষ্কার করতে হয়। PostgreSQL-এ এটি `VACUUM`, এবং একটি দীর্ঘস্থায়ী transaction যা cleanup আটকে রাখে তা table bloat ঘটায় — একটি বাস্তব এবং সাধারণ production সমস্যা যা প্রথমবার team-দের অবাক করে দেয়।
- **একই level-এর নাম ভিন্ন জিনিস বোঝায়।** PostgreSQL-এর Repeatable Read কিছু anomaly প্রতিরোধ করে যা MySQL-এরটি করে না; একটি engine-এর "Repeatable Read" আরেকটির মতো একই guarantee নয়। পোর্টেবল ধারণা অনিরাপদ।
- **এর কোনোটিই একাধিক database জুড়ে কাজ করে না।** Isolation প্রতিটি database-এর নিজস্ব বিষয়। কোনো operation যত তাড়াতাড়ি service বা shard জুড়ে ছড়িয়ে পড়ে, ততই আপনি সেই সাগা (saga) ও idempotency-র জগতে ফিরে যান।

## When You Need It — and When You Don't

| যখন isolation বাড়ান বা স্পষ্টভাবে lock নিন | যখন default-ই যথেষ্ট |
|---|---|
| টাকা, inventory, সিট, বা quota জড়িত থাকলে | operation-টি একটি সরল insert বা একটি single-row read হলে |
| logic-টি read-check-write হলে | read-গুলো শুধু প্রদর্শনের জন্য এবং সামান্য staleness ঠিক থাকলে |
| একটি uniqueness বা capacity invariant বজায় রাখতে হলে | write-টি স্বাভাবিকভাবেই idempotent এবং order-independent হলে |
| একটি report-কে internally consistent হতে হলে | contention সত্যিই কম এবং ঝুঁকিও কম হলে |

**সবচেয়ে ভালো প্রথম পদক্ষেপ:** read-check-write-কে একটি একক atomic statement হিসেবে পুনর্লিখন করুন অথবা একটি database constraint-এর ওপর নির্ভর করুন। এটি যেকোনো isolation level আলোচনার চেয়ে সস্তা এবং নিরাপদ।

## Why This Shows Up in Interviews

এটি একটি senior-level পার্থক্যকারী বিষয়। ticket booking, seat reservation, inventory, বা account balance জড়িত prompt আসলে ছদ্মবেশে concurrency প্রশ্ন, এবং interviewer অপেক্ষা করছে দেখার জন্য আপনি সেটা লক্ষ্য করেন কিনা। এটা বলা যে "দুজন user হয়তো একই সাথে শেষ সিটটি বুক করতে পারে, তাই আমি হয় `SELECT FOR UPDATE` দিয়ে একটি row lock নেব, অথবা একটি version column দিয়ে optimistic concurrency ব্যবহার করব, অথবা এটিকে একটি conditional atomic update বানাব — এবং এই কারণে আমি একটি বেছে নেব" একটি খুব শক্তিশালী উত্তর। MVCC কী, এবং reader-রা writer-দের block করে না — এটা জানা প্রমাণ করে আপনি SQL-এর নিচে আসলে কী ঘটছে তা বোঝেন।

## How It Connects

এটিই **ACID** (topic 16)-এর "I", গভীরভাবে পরীক্ষা করা হলো। এটি topic 29-এর **consistency model**-এর single-node সমতুল্য — সেই একই মূল টানাপোড়েন correctness এবং concurrency-র মাঝে, একটি একটি database-এর ভেতরে, আরেকটি একটি network জুড়ে। এটি **shard** (topic 14) এবং **microservices** (topic 30)-এ কাজ করা বন্ধ করে দেয়, এবং এই কারণেই **distributed transaction** (topic 28) এবং **distributed locking** (topic 40) বিদ্যমান। MVCC-এর implementation **storage engine internals** (topic 38)-এর সাথে সংযুক্ত, এবং এটি **ride-sharing** (topic 54) এবং booking-স্টাইল কেস স্টাডিতে নির্ণায়ক।

**পরবর্তী:** [LSM Trees vs B-Trees](../38-lsm-trees-vs-b-trees-storage-engine-internals/why.md) — আরও এক স্তর গভীরে, row-গুলো আসলে কীভাবে ডিস্কে লেখা হয় তার মধ্যে।
