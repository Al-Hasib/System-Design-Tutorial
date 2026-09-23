# এই বিষয়টি কেন গুরুত্বপূর্ণ: Rate Limiting Algorithms

> **এক বাক্যে:** ইন্টারনেটের জন্য উন্মুক্ত যেকোনো endpoint শেষ পর্যন্ত আপনার পরিকল্পনার চেয়ে অনেক বেশি চাপ পাবে — একজন আক্রমণকারী, একটি ত্রুটিপূর্ণ client, বা একজন উৎসাহী কাস্টমারের কারণে — এবং একটি limiter ছাড়া একমাত্র জিনিস যা এটা থামায় তা হলো আপনার সিস্টেমের ভেঙে পড়া।

## এই ধারণার আগে দুনিয়া কেমন ছিল (The World Before This Idea)

কোনো rate limit নেই। এখন কী কী সম্ভব তা ভাবুন:

- একটি script আপনার login endpoint-এর বিরুদ্ধে প্রতি মিনিটে 50,000টি পাসওয়ার্ড চেষ্টা করে।
- একজন কাস্টমারের retry loop-এ কোনো backoff নেই, তাই একটি ক্ষণস্থায়ী error স্থায়ীভাবে প্রতি সেকেন্ডে 10,000টি request-এ পরিণত হয়।
- একটি scraper আপনার পুরো catalog টেনে নেয়, সমস্ত প্রকৃত user মিলিয়ে যতটুকু capacity ব্যবহার করে তার চেয়েও বেশি ব্যবহার করে।
- কেউ একরাতে 100,000টি ভুয়া account সাইন আপ করে।
- একটি একক ব্যয়বহুল API কল (একটি report, একটি search, একটি export) loop-এ কল করা হয় এবং বাকি সবকিছুকে ক্ষুধার্ত রাখে।

এগুলোর কোনোটাতেই দুষ্টুমির প্রয়োজন নেই — বাগযুক্ত-client-এর ঘটনা যথেষ্ট ব্যবধানে সবচেয়ে সাধারণ — এবং এগুলো সবগুলোর ফলাফল একই: একজন caller এমন capacity ব্যবহার করে ফেলে যা সবার জন্য বরাদ্দ, এবং সিস্টেম সব user-এর জন্য অবনতি হয় বা মারা যায়।

## এটি যে সমস্যাগুলো সমাধান করে (The Problems It Solves)

### ১. একজন caller-এর কারণে সবার জন্য service অবনতি হওয়া
**আপনি যা দেখবেন:** সর্বত্র latency spike এবং error; তদন্তে দেখা যায় একটি একক API key traffic-এর 80%-এর জন্য দায়ী।

**কেন এটা ঘটে:** Capacity শেয়ারড এবং অবরাদ্দকৃত। কোনো limit ছাড়া, first-come-first-served মানে যে দ্রুততম চায় সে সবকিছু পেয়ে যায়।

**Rate limiting এটা কীভাবে সমাধান করে:** প্রতি-client quota একটি শেয়ারড resource-কে বরাদ্দকৃত অংশে রূপান্তরিত করে। একজন client তার limit-এ পৌঁছালে 429 পায়; বাকি সবাই অপ্রভাবিত থাকে। এই fairness বৈশিষ্ট্যটি — খাঁটি protection নয় — এটাই দৈনন্দিন মূল্য।

### ২. Brute force এবং enumeration
**আপনি যা দেখবেন:** Login-এর বিরুদ্ধে credential stuffing, প্রমোশন কোড নিঃশেষে অনুমান করা, ব্যক্তিগত তথ্য scrape করার জন্য user ID enumerate করা।

**কেন এটা ঘটে:** এই আক্রমণগুলো সম্পূর্ণভাবে সস্তায় বিশাল সংখ্যক প্রচেষ্টা করতে পারার উপর নির্ভরশীল।

**Rate limiting এটা কীভাবে সমাধান করে:** এটা প্রতিটি প্রচেষ্টার সময়কে ব্যয়বহুল করে তোলে। প্রতি account প্রতি মিনিটে পাঁচটি login প্রচেষ্টা একটি 50,000-প্রচেষ্টার আক্রমণকে দুই মিনিটের কাজ থেকে অসম্ভব কাজে পরিণত করে। নিরাপত্তা-সংবেদনশীল endpoint-এর জন্য, rate limiting-ই *হলো* নিয়ন্ত্রণ।

### ৩. Cost amplification
**আপনি যা দেখবেন:** একটি SMS provider, একটি LLM API, বা cloud egress থেকে একটি বিস্ময়কর পাঁচ-অঙ্কের বিল।

**কেন এটা ঘটে:** আপনি যে প্রতিটি কল ফরওয়ার্ড করেন তার জন্য প্রকৃত অর্থ খরচ হয়, এবং একজন সীমাহীন caller মানে একটি সীমাহীন বিল।

**Rate limiting এটা কীভাবে সমাধান করে:** Limit একটি ব্যয়ের সিলিং হয়ে যায়। এটা ভেতরের দিকে প্রযোজ্য (আপনার user থেকে নিজেকে রক্ষা করা) এবং বাইরের দিকে (আপনার নিজের retry storm একটি paid third-party API-তে আঘাত করা থেকে নিজেকে রক্ষা করা)।

### ৪. Burst আচরণ যা একটি সরল limiter ভুল করে
**আপনি যা দেখবেন:** প্রতি মিনিটে 100-এর একটি fixed-window limiter একটি client-কে 11:59:59-এ 100টি request পাঠাতে দেয় এবং 12:00:00-এ আরও 100টি — এক সেকেন্ডে 200টি request, উদ্দিষ্ট হারের দ্বিগুণ, ঠিক সেই boundary-তে।

**কেন এটা ঘটে:** Fixed window হঠাৎ করে রিসেট হয়, তাই boundary-টি শোষণযোগ্য।

**অ্যালগরিদমগুলো এটা কীভাবে সমাধান করে:** এই কারণেই অ্যালগরিদম নির্বাচন গুরুত্বপূর্ণ। **Token bucket** নিয়ন্ত্রিত burst-এর অনুমতি দেয় (token একটি cap পর্যন্ত জমা হয়) দীর্ঘমেয়াদী গড়কে সীমাবদ্ধ রাখার পাশাপাশি — সাধারণত সেরা ডিফল্ট কারণ বাস্তব traffic burst-প্রবণ এবং user রা একটি বৈধ burst-এ throttle হওয়া অপছন্দ করে। **Leaky bucket** একটি কঠোরভাবে মসৃণ output rate প্রয়োগ করে, যা আপনি চান যখন একটি fixed capacity সহ downstream সিস্টেম রক্ষা করছেন। **Sliding window log** নিখুঁত কিন্তু memory-ক্ষুধার্ত; **sliding window counter** সস্তায় এটার আনুমানিক করে এবং boundary সমস্যা ঠিক করে। প্রতিটিই একটি ভিন্ন প্রশ্নের সঠিক উত্তর।

## আপনাকে যে মূল্য দিতে হবে (The Price You Pay)

- **বৈধ user রা ব্লক হয়ে যায়।** খুব কম সেট করা limit বাস্তব workflow ভেঙে দেয়, এবং ফলস্বরূপ support load একটি প্রকৃত খরচ। Tiered limit, burst allowance, এবং স্পষ্ট `Retry-After` header এভাবেই আপনি এটা প্রশমিত করেন।
- **Distributed counting একটি প্রকৃত সমস্যা।** দশটি API server নিয়ে, প্রতিটি স্থানীয়ভাবে গণনা করলে, আপনার "প্রতি মিনিটে 100" limit আসলে প্রতি মিনিটে 1000। এটাকে সঠিক করতে প্রতিটি request-এর hot path-এ শেয়ারড state (Redis) প্রয়োজন — যা latency এবং একটি critical dependency যোগ করে। বিকল্পগুলো (1/N-এ local limit, বা approximate sync) accuracy-র বিনিময়ে গতি দেয়, এবং সেই ট্রেড-অফটি সচেতনভাবে করা উচিত।
- **Client চিহ্নিত করা যতটা সহজ মনে হয় তার চেয়ে কঠিন।** IP-ভিত্তিক limiting একটি কর্পোরেট NAT বা mobile carrier gateway-এর পেছনে থাকা সবাইকে শাস্তি দেয়, এবং একটি proxy pool দিয়ে সহজেই এড়ানো যায়। API key ভালো কিন্তু limiting-এর আগে authentication প্রয়োজন — যার মানে unauthenticated endpoint-এর জন্য একটি ভিন্ন কৌশল দরকার।
- **Limiter নিজেই ব্যর্থ হতে পারে।** যদি Redis down থাকে, আপনি কি fail open করবেন (কোনো limit নেই, overload-এর ঝুঁকি) নাকি fail closed করবেন (সবকিছু প্রত্যাখ্যান, একটি outage ঘটায়)? এখানে সার্বজনীনভাবে সঠিক কোনো উত্তর নেই; শুধু সেই উত্তর আছে যা আপনি আগে থেকে সচেতনভাবে বেছে নিয়েছেন।
- **এটা DDoS protection নয়।** একটি volumetric আক্রমণ আপনার application-layer limiter চলার আগেই আপনার network পরিপূর্ণ করে ফেলে। সেটার জন্য upstream filtering দরকার।

## কখন এটা প্রয়োজন — এবং কখন নয় (When You Need It — and When You Don't)

| Rate limit করুন যখন | এটা কম গুরুত্বপূর্ণ যখন |
|---|---|
| Endpoint টি publicly reachable | এটা একটি trusted network-এর পেছনে internal-only |
| Request গুলো ব্যয়বহুল (compute, money, third-party কল) | Operation টি তুচ্ছভাবে সস্তা এবং idempotent |
| এটা নিরাপত্তা-সংবেদনশীল (login, password reset, signup, OTP) | — |
| আপনি tiered plan অফার করেন এবং quota enforcement দরকার | — |
| আপনি একটি ভঙ্গুর downstream dependency রক্ষা করছেন | — |

## ইন্টারভিউতে এটা কেন আসে (Why This Shows Up in Interviews)

Rate limiting বেশিরভাগ ডিজাইনে একটি component (এটা মূলত প্রতিটি diagram-এ API gateway-তে থাকে) এবং একটি সম্পূর্ণ স্বতন্ত্র ডিজাইন প্রশ্নও — এই কোর্সের topic 49 ঠিক সেটাই। ইন্টারভিউয়াররা প্রশ্ন করেন: কোন অ্যালগরিদম এবং কেন, counter state কোথায় থাকে, এটা অনেক server জুড়ে কীভাবে কাজ করে, client-কে কী ফেরত দেওয়া হয় (429 এবং `Retry-After`), এবং rate-limiting store অনুপলব্ধ হলে কী হয়। Distributed-counting সমস্যাটি মূল বিষয়, এবং যেসব প্রার্থী সরাসরি সেখানে চলে যান তারা প্রমাণ করেন যে তারা single-server সংস্করণের বাইরে চিন্তা করেছেন।

## এটি কীভাবে সংযুক্ত (How It Connects)

Rate limiting **API gateway**-তে (topic 9) বা **reverse proxy**-তে (topic 8) থাকে, প্রায় সবসময় শেয়ারড atomic counter-এর জন্য **Redis**-এ (topic 19) প্রয়োগ করা হয়, এবং একটি মূল **security** নিয়ন্ত্রণ (topic 36)। এটা **circuit breaker**-দের (topic 26) পরিপূরক — একটি limiter আপনাকে caller থেকে রক্ষা করে, একটি breaker আপনাকে callee থেকে রক্ষা করে। এটা **Design a Rate Limiter** কেস স্টাডির (topic 49) বিষয়বস্তু।

**পরবর্তী:** [Circuit Breaker, Retry & Bulkhead Patterns](../26-circuit-breaker-retry-and-bulkhead-patterns/why.md) — আপনি যে service-গুলোর উপর নির্ভরশীল সেগুলো থেকে নিজেকে রক্ষা করা।
