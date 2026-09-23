# এই Topic কেন গুরুত্বপূর্ণ: Batch Processing বনাম Stream Processing

> **এক বাক্যে:** কিছু উত্তর আগামীকাল পর্যন্ত অপেক্ষা করার যোগ্য, আর কিছু উত্তর এক মিনিট দেরিতে এলেই মূল্যহীন হয়ে যায় — ভুল processing model বেছে নেওয়া মানে হয় প্রয়োজন নেই এমন infrastructure-এ টাকা পোড়ানো, নয়তো এমন insight পাঠানো যা গুরুত্বপূর্ণ থাকার সময় পার হয়ে যাওয়ার পর আসে।

## এই ধারণার আগের পৃথিবী

দুটো বিপরীত failure mode, দুটোই সাধারণ:

**Streaming দরকার ছিল যেখানে Batch ব্যবহার হয়েছে।** Fraud detection একটা রাতের job হিসেবে চলে। একটা চুরি হওয়া card সকাল ৯টায় ব্যবহার হয় এবং পরদিন রাত ২টায় ফ্ল্যাগ হয় — সতেরো ঘণ্টা fraudulent purchase-এর পর। Model accurate এবং pipeline ভালোভাবে engineer করা। তবুও এটা অকেজো, কারণ উত্তরটা সেই একমাত্র মুহূর্তের অনেক পরে আসে যখন এর উপর কিছু করা যেত।

**Batch দরকার ছিল যেখানে Streaming ব্যবহার হয়েছে।** একটা team মাসিক financial report গণনার জন্য একটা real-time pipeline তৈরি করে। এটা ক্রমাগত চলে, একটা রাতের job-এর চেয়ে দশগুণ বেশি খরচ করে, debug করা অনেক কঠিন, এবং যখনই একটা দেরিতে আসা event আসে তখন সংখ্যাগুলো পুনরায় বিবৃত হয় — এমন একটা report-এর জন্য যা মাসের পাঁচ তারিখের আগে কেউ পড়েই না।

দুটো team-ই দক্ষ ছিল। দুটোই business-এর প্রকৃতপক্ষে প্রয়োজনীয় *decision latency*-র সাথে processing model মেলাতে ব্যর্থ হয়েছে।

## এটা যেসব সমস্যার সমাধান করে

### ১. এমন insight যা সিদ্ধান্ত নেওয়ার মুহূর্ত পার হওয়ার পর আসে
**আপনি যা দেখবেন:** Alert, recommendation, এবং detection যা technically সঠিক কিন্তু practically অপ্রাসঙ্গিক।

**কেন এটা ঘটে:** Batch job-এর latency-র একটা floor আছে যা তার schedule interval, plus তার runtime-এর সমান। একটা daily job কখনোই ২৪ ঘণ্টার কমে কিছু বলতে পারে না।

**Stream processing কীভাবে এটা সমাধান করে:** Events যখনই আসে তখনই প্রসেস করা হয়, তাই ফলাফল সেকেন্ডের মধ্যে পাওয়া যায়। Fraud, abuse detection, live dashboard, dynamic pricing, এবং trending content-এর জন্য, এটাই একটা কাজ করা এবং না-করা system-এর মধ্যে পার্থক্য।

### ২. এক সপ্তাহ লাগে এমন Reprocessing
**আপনি যা দেখবেন:** আপনি একটা metric গণনায় একটা bug খুঁজে পান। এটা ঠিক করতে দুই বছরের history পুনরায় গণনা করতে হবে, এবং এটা করার কোনো mechanism নেই।

**কেন এটা ঘটে:** Streaming systems একটা window-এর উপর incremental-ভাবে গণনা করে এবং প্রায়ই raw input ধরে রাখে না।

**Batch কীভাবে এটা সমাধান করে:** Batch একটা সম্পূর্ণ, bounded, সংরক্ষিত dataset-এর উপর কাজ করে। কোড ঠিক করুন, job পুনরায় চালান, সংশোধিত ফলাফল পান — একসাথে সব ডেটায় পূর্ণ access সহ, যা এমন গণনাও সম্ভব করে (global sorts, full joins, model training) যা streaming efficiently করতে পারে না।

### ৩. সময়ের কারণে সৃষ্ট Correctness সমস্যা
**আপনি যা দেখবেন:** একটা mobile app offline অবস্থায় events buffer করে এবং তিন ঘণ্টা পর upload করে। সেই window-এর জন্য আপনার hourly count ভুল, এবং সেগুলো ইতিমধ্যেই publish হয়ে গেছে।

**কেন এটা ঘটে:** একটা stream-এ, একটা event যখন *ঘটেছে* এবং যখন *এসে পৌঁছেছে* তার সময় ভিন্ন হতে পারে, কখনো কখনো অনেকটাই। যেকোনো windowed aggregate-কে সিদ্ধান্ত নিতে হয় stragglers-এর জন্য কতক্ষণ অপেক্ষা করবে।

**এই topic কীভাবে এটা সমাধান করে:** এটা event time বনাম processing time, watermarks, এবং late-arrival handling-কে surprise না করে স্পষ্ট design সিদ্ধান্ত বানায়। Batch গণনা করার আগে window বন্ধ হওয়ার জন্য অপেক্ষা করে এই সমস্যা এড়িয়ে যায় — যা কারণ কেন এটা সহজ এবং কারণ কেন এটা ধীর তার একটা অংশ।

### ৪. Batch সমস্যার জন্য Streaming-এর দাম দেওয়া
**আপনি যা দেখবেন:** একটা সবসময়-চালু cluster ২৪/৭ চলছে এমন সংখ্যা তৈরি করার জন্য যা দিনে একবার ব্যবহার হয়।

**কেন এটা ঘটে:** "Real-time" শুনতে একেবারে ভালো মনে হয়, তাই এটা default হিসেবে বেছে নেওয়া হয়।

**এই topic কীভাবে এটা সমাধান করে:** এটা পছন্দটাকে একটা একক প্রশ্নের চারপাশে পুনর্গঠন করে — *এর উপর কারও কত দ্রুত কাজ করা দরকার?* — যা সাধারণত প্রকাশ করে যে অনেকখানি ডেটা কাজ আসলে অনেক কম খরচ এবং জটিলতায় একটা scheduled job হিসেবে ঠিকই চলতে পারে।

## যে দাম আপনাকে দিতে হবে

**Streaming-এর খরচ:**
- সবসময়-চালু infrastructure, এবং এর সাথে আসা operational burden।
- Stateful operations (windows, joins, aggregations)-এর জন্য managed state, checkpointing, এবং recovery প্রয়োজন — এখানেই বেশিরভাগ প্রকৃত complexity থাকে।
- Exactly-once semantics achievable কিন্তু দাবিদার; at-least-once plus idempotency হলো practical norm।
- একটা unbounded stream debug করা একটা fixed input-এ পুনরায় চালানো যায় এমন job debug করার চেয়ে সত্যিকারভাবে কঠিন।
- Backpressure: যদি producers, consumers-এর চেয়ে দ্রুত হয়, তাহলে কিছু একটাকে ছাড় দিতেই হবে।

**Batch-এর খরচ:**
- Latency সবসময় schedule-এর সমান।
- Bursty resource usage — ঘণ্টার পর ঘণ্টা idle, তারপর একটা বিশাল spike।
- একটা ব্যর্থ job মানে হতে পারে সারাদিন কোনো ডেটা নেই, এবং পরের window-এর আগে পুনরায় চালানো সম্ভব নাও হতে পারে।
- ফলাফল স্বভাবগতভাবেই stale, এবং users-কে এটা বুঝতে হবে।

**আর hybrid-এরও খরচ আছে।** Lambda architecture (একটা batch এবং একটা streaming path দুটোই চালানো) মানে একই business logic দুই সিস্টেমে দুইবার maintain করা এবং তাদের মতভেদ মেলানো। Kappa architecture (শুধু streaming, reprocessing-এর জন্য log থেকে replay করা) duplication এড়ায় কিন্তু একটা durable, replayable log এবং সবকিছুকে reprocessable রাখার discipline দরকার।

## কখন এটা আপনার দরকার — এবং কখন দরকার নেই

| Stream যখন | Batch যখন |
|---|---|
| সিদ্ধান্ত সেকেন্ডের মধ্যে নিতে হয় (fraud, abuse, alerting) | ফলাফল দৈনিক বা তার চেয়ে কম ঘন ঘন ব্যবহার হয় |
| Users live-updating value দেখে | পুরো dataset-এর উপর global operation দরকার |
| ডেটা ক্রমাগত এবং unbounded | একটা fix-এর পর history পুনরায় প্রসেস করা দরকার |
| Late data, approximate data-র চেয়ে খারাপ | Accuracy এবং completeness, freshness-এর চেয়ে ভালো |
| — | Cost efficiency গুরুত্বপূর্ণ এবং latency নয় |

## এটা কেন Interview-এ আসে

Analytics, metrics, recommendations, এবং trending features বেশিরভাগ বড় design prompt-এ দেখা যায়। Interviewer-রা যে signal চায় তা হলো আপনি architecture বেছে নেওয়ার আগে *decision latency* নিয়ে প্রশ্ন করছেন কিনা — "এই সংখ্যাটা কতটা fresh হওয়া দরকার?" — এবং তারপর সেই অনুযায়ী বেছে নিচ্ছেন, দেখতে চিত্তাকর্ষক লাগে বলে default-ভাবে real-time বেছে না নিয়ে। যেসব candidates streaming-এর কঠিন অংশগুলোও নাম করতে পারে (event time বনাম processing time, windowing, late arrivals, state management), তারা কেবল Kafka এবং Flink-এর নাম বলার চেয়ে অনেক গভীর জ্ঞান প্রদর্শন করছে।

## এটা কীভাবে সম্পর্কযুক্ত

দুটো মডেলই একই **message queue** infrastructure (topic 20) এবং একটা event-driven system দ্বারা তৈরি **events** (topic 22) থেকে consume করে। Streaming systems bounded memory-তে counts এবং cardinalities আনুমানিক করতে **probabilistic data structures** (topic 42)-এর উপর অনেকখানি নির্ভর করে, এবং redelivery থেকে বাঁচার জন্য **idempotency** (topic 29)-এর উপর নির্ভর করে। Freshness-বনাম-accuracy trade হলো Module 3-এর **consistency** trade-off-এরই আরেকটা উদাহরণ, এবং **logical clocks** (topic 41) event ordering সম্পর্কে চিন্তাভাবনার ভিত্তি তৈরি করে।

**পরবর্তী:** [Consistent Hashing Explained](../../Module-06-Distributed-Systems-Concepts/24-consistent-hashing-explained/why.md) — cluster পরিবর্তন হওয়ার প্রতিবার সবকিছু পুনরায় সাজানো ছাড়াই কীভাবে node-গুলোর মধ্যে ডেটা ছড়িয়ে দেওয়া যায়।
