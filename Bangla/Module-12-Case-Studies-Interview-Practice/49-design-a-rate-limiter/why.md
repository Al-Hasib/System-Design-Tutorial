# Why This Topic Matters: Design a Rate Limiter

> **এক বাক্যে:** single-server version টি বিশ লাইন কোড, এবং ঠিক এই কারণেই এই সমস্যাটি জিজ্ঞাসা করা হয় — আকর্ষণীয় সবকিছু তখনই দেখা যায় যখন আপনার একের বেশি server থাকে, এবং সেখান থেকেই interview আসলে শুরু হয়।

## Why This Case Study Exists

বেশিরভাগ design prompt একটি product তৈরি করা নিয়ে। এটি একটি *component* তৈরি করা নিয়ে — এমন একটি যা প্রতিটি request-এর hot path-এ বসে থাকে, এতে প্রায় কোনো latency যোগ করা যাবে না, concurrency-র অধীনে সঠিক হতে হবে, এবং এর নিজস্ব dependency ব্যর্থ হলেও কাজ চালিয়ে যেতে হবে।

এটি distributed system-এর প্রকৃতি সম্পর্কেও অস্বাভাবিকভাবে সৎ। naive সমাধানটি একটি machine-এ পুরোপুরি কাজ করে। দশটি machine-এ scale করুন এবং এটি নীরবে দশ গুণ ভুল হয়ে যায়, কোনো error বা alert ছাড়াই। "স্পষ্টতই সঠিক" এবং "scale-এ নীরবে ভাঙা"-র মধ্যেকার সেই ফাঁকটাই হলো শিক্ষণীয় বিষয়।

## The Design Problems It Forces You to Solve

### ১. একাধিক server জুড়ে সঠিকভাবে count করা
**সমস্যা:** দশটি API server, প্রতিটি local memory-তে "প্রতি ব্যবহারকারী প্রতি মিনিটে requests" track করছে। আপনার ১০০/মিনিট limit আসলে ১০০০/মিনিট।

**কেন এটি কঠিন:** সঠিকতার জন্য shared state দরকার, এবং প্রতিটি request-এর hot path-এ shared state latency খরচ করে এবং একটি dependency তৈরি করে।

**আপনি যা শিখবেন:** বিকল্পগুলো এবং তাদের প্রকৃত trade-off। **Redis-এ centralized counter** সঠিক এবং একটি network round trip এবং একটি critical dependency যোগ করে। **limit-এর 1/N-এ local counter** দ্রুত এবং ভুল হয় যখন server জুড়ে load অসম হয়। **periodic synchronization সহ local counter** একটি মাঝামাঝি সমাধান যা মোটামুটি সঠিক এবং eventually consistent। Redis-ই সবকিছুর সমাধান ধরে নেওয়ার বদলে trade-off-টিকে নাম দেওয়াই মূল বিষয় — এবং আপনি যদি Redis বেছে নেন, তাহলে জানা দরকার যে increment-and-check অবশ্যই atomic হতে হবে (একটি Lua script অথবা `EXPIRE`-সহ একটি pipelined `INCR`), read-then-write race নয়।

### ২. প্রকৃত traffic shape-এর জন্য একটি algorithm বেছে নেওয়া
**সমস্যা:** ১০০/মিনিটের একটি fixed-window counter একটি window boundary জুড়ে এক সেকেন্ডে ২০০টি request অনুমতি দেয়।

**কেন এটি কঠিন:** প্রতিটি algorithm-এর একটি স্বতন্ত্র failure mode আছে, এবং সঠিকটি নির্ভর করে আপনি কী রক্ষা করছেন এবং বৈধ traffic কতটা bursty তার উপর।

**আপনি যা শিখবেন:** **Token bucket** long-run rate সীমাবদ্ধ রাখার সময় bounded burst অনুমতি দেয় — সাধারণত সেরা default, কারণ প্রকৃত client traffic bursty হয় এবং একটি বৈধ burst-কে throttle করা support ticket তৈরি করে। **Leaky bucket** কঠোরভাবে smooth output rate কার্যকর করে, যা আপনি চান যখন downstream-এর জিনিসটির fixed capacity থাকে। **Sliding window log** সঠিক এবং প্রতি request-এ একটি timestamp সংরক্ষণ করে, যা scale-এ অনেক বেশি memory। **Sliding window counter** log-কে সস্তায় approximate করে এবং boundary সমস্যা ঠিক করে। একটি বেছে নিতে পারা এবং traffic pattern-এর বিরুদ্ধে সেটাকে যুক্তিসঙ্গত করা পারাটাই grade করা হয়।

### ৩. limiter নিজেই ব্যর্থ হলে কী করবেন তা ঠিক করা
**সমস্যা:** Redis unavailable। আপনি কি সব traffic-কে অনুমতি দেবেন নাকি সব traffic reject করবেন?

**কেন এটি কঠিন:** উভয় উত্তরই খারাপ। Fail open করা মানে একটি incident-এর সময় একটি অরক্ষিত system, ঠিক যখন এটি সবচেয়ে ভঙ্গুর। Fail closed করা মানে আপনার rate limiter একটি সম্পূর্ণ outage ঘটায়।

**আপনি যা শিখবেন:** এটি একটি ইচ্ছাকৃত product সিদ্ধান্ত, technical সিদ্ধান্ত নয়, এবং এটি সাধারণত প্রতিটি endpoint অনুযায়ী ভিন্ন হয় — সাধারণ API traffic-এর জন্য fail open (availability perfect limit-এর চেয়ে বেশি গুরুত্বপূর্ণ), login এবং payment endpoint-এর জন্য fail closed (যেখানে limit-টিই *হলো* security control)। এখানে একটি অবস্থান থাকা, এবং এটি যে পরিবর্তিত হয় তা জানা, একটি শক্তিশালী সংকেত।

### ৪. client-কে identify করা, এবং তাদের বলা কী হয়েছে
**সমস্যা:** IP দ্বারা limit করা একটি corporate NAT বা mobile carrier gateway-র পেছনের সবাইকে শাস্তি দেয় এবং একটি proxy pool দিয়ে সহজেই এড়ানো যায়। API key দ্বারা limit করার জন্য প্রথমে authentication দরকার, যা unauthenticated endpoint-এর নেই।

**আপনি যা শিখবেন:** Layered identity — যেখানে পাওয়া যায় সেখানে API key, যেখানে authenticated সেখানে user ID, উচ্চতর threshold সহ একটি coarse fallback হিসেবে IP — এবং প্রতিটি tier-এর জন্য ভিন্ন limit। এছাড়াও client-facing contract: `429 Too Many Requests` return করুন `Retry-After` সহ, এবং আদর্শভাবে `X-RateLimit-Limit`, `X-RateLimit-Remaining`, এবং `X-RateLimit-Reset` যাতে ভালো-আচরণকারী client গুলো নিজেদের throttle করতে পারে আপনাকে একটি retry storm-এ ঠেলে দেওয়ার পরিবর্তে।

## What It Costs to Get Wrong

- **শুধুমাত্র single-server version-এর উত্তর দেওয়া** এবং distributed সমস্যা না তোলা এই interview-টি খারাপভাবে যাওয়ার সবচেয়ে সাধারণ উপায়।
- **counter-এ read-then-write** একটি race যা ঠিক তখনই under-count করে যখন traffic সবচেয়ে বেশি থাকে।
- **যুক্তি ছাড়া একটি algorithm বেছে নেওয়া** memorization হিসেবে পড়া হয়।
- **limiter-এর failure mode উপেক্ষা করা** প্রতিটি request path-এ একটি component রেখে দেয় যার একটি incident-এর সময় undefined আচরণ থাকে।
- **অতিরিক্ত সীমাবদ্ধ করা** এমন একটি limiter তৈরি করে যা technically সঠিক কিন্তু বাণিজ্যিকভাবে ক্ষতিকর, কারণ বৈধ burst গুলো স্বাভাবিক ব্যবহারের অংশ।

## Why Interviewers Choose This One

এটি শেষ করার মতো যথেষ্ট compact এবং গভীরে যাওয়ার মতো যথেষ্ট গভীর। এটি constraint-এর অধীনে algorithm selection, distributed state management, atomic operations, failure-mode reasoning, এবং API design পরীক্ষা করে — পাঁচটি জিনিস, একটি প্রশ্নে, কোনো domain knowledge ছাড়াই। এটি এমন একটি component যা প্রায় প্রতিটি বাস্তব system-এ বিদ্যমান, তাই আলোচনা hypothetical না হয়ে concrete থাকে।

## How It Connects

এটি হলো **rate limiting algorithms** (topic 25)-এর প্রয়োগকৃত version, atomic operations সহ **Redis**-এ implement করা (topic 19), **API gateway** বা **reverse proxy**-তে deploy করা (topics 9, 8)। এটি brute force-এর বিরুদ্ধে একটি মূল **security** control (topic 36), call-এর অন্য পাশে **circuit breakers**-কে (topic 26) পরিপূরক করে, এবং খুব উচ্চ key cardinality-তে এটি **Count-Min Sketch**-এর (topic 42) দিকে ফিরে যায়। fail-open/fail-closed প্রশ্নটি **CAP** (topic 15) থেকে **availability versus correctness** trade-off-এর একটি সরাসরি উদাহরণ।

**পরবর্তী:** [Design a Chat Application](../50-design-a-chat-application-whatsapp/why.md) — যেখানে connection state এবং delivery guarantee গুলো পুরো সমস্যা হয়ে ওঠে।
