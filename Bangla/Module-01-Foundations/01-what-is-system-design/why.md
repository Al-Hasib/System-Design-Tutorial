# এই Topic কেন গুরুত্বপূর্ণ: System Design কী?

> **এক বাক্যে:** System design হলো একটি discipline যেখানে code লেখার আগেই *অংশগুলো কীভাবে একসাথে খাপ খায়* তা ঠিক করা হয় — আর এটা এড়িয়ে যাওয়াই কারণ কেন laptop-এ কাজ করা systems production-এ ভেঙে পড়ে।

## এই Idea আসার আগের পৃথিবী

একজন developer একটি ticket পান, editor খোলেন, এবং একটি feature লেখেন। এটা কাজ করে। আরও দশটি feature আসে; প্রতিটি একইভাবে লেখা হয়। আঠারো মাস পর কেউই ব্যাখ্যা করতে পারে না system কীভাবে কাজ করে, একটি মাত্র slow query checkout বন্ধ করে দেয়, এবং একটি নতুন feature যোগ করতে হলে নয়টি এমন file ছুঁতে হয় যা কেউ বোঝে না।

সেই গল্পে কিছুই একটি *coding* failure নয়। প্রতিটি individual function ঠিকঠাক ছিল। যা অনুপস্থিত ছিল তা হলো এই ধরনের প্রশ্নের একটি সুচিন্তিত উত্তর: একসাথে ১০,০০০ মানুষ এটা করলে কী হবে? এই data কোথায় থাকে? সেই server নষ্ট হয়ে গেলে কী ভেঙে পড়বে? সেই প্রশ্নগুলোই system design।

## এটি যেসব সমস্যার সমাধান করে

### ১. "development-এ কাজ করেছিল" — কিন্তু আর কোথাও নয়
**আপনি যা দেখেন:** একটি feature যা ৫০টি test row নিয়ে ৪০ ms-এ ফেরত আসে, তা ৫০ লক্ষ production row নিয়ে ৩০ সেকেন্ড সময় নেয়। একটি service যা একজন user-কে নিখুঁতভাবে handle করে, তা ২০০ concurrent user-এ ভেঙে পড়ে।

**কেন এটা ঘটে:** Local development-এ কোনো latency, কোনো concurrency, কোনো failure, এবং কোনো data volume থাকে না। যেসব শক্তি প্রকৃতপক্ষে একটি production system গঠন করে তার কোনোটিই উপস্থিত থাকে না, তাই এসব বিষয় না ভেবে লেখা code দুর্ঘটনাক্রমে এমন একটি পৃথিবীর জন্য tune করা হয় যা বাস্তবে নেই।

**System design কীভাবে এটি সমাধান করে:** এটা আপনাকে বাধ্য করে operating conditions *আগে থেকেই* স্পষ্ট করতে — প্রত্যাশিত traffic, data size, latency budget, failure modes — এবং তারপর এমন design বেছে নিতে যা সেই conditions-এর অধীনে টিকে থাকে, বরং একটি outage-এর সময় সেই mismatch আবিষ্কার করার পরিবর্তে।

### ২. Rebuilds যা এক বছর সময় নেয়
**আপনি যা দেখেন:** "আমাদের পুরো জিনিসটা নতুন করে লিখতে হবে।" Database-কে shard করা যায় না কারণ প্রতিটি table প্রতিটি অন্য table-কে reference করে। Scale করতে হলে redesign দরকার, শুধু একটি config change নয়।

**কেন এটা ঘটে:** প্রাথমিক structural choices — সবকিছুর জন্য একটি database, সর্বত্র synchronous calls, server memory-তে সংরক্ষিত state — করা সস্তা কিন্তু পাল্টানো ভীষণ ব্যয়বহুল। এগুলো জমাট বাঁধে কারণ এরপর যা কিছু তৈরি হয় তার সবই এগুলোর উপর নির্ভরশীল থাকে।

**System design কীভাবে এটি সমাধান করে:** এটা চিহ্নিত করে কোন decisions *পাল্টানো কঠিন* (data model, service boundaries, consistency guarantees), এবং সেখানে চিন্তাভাবনার সময় ব্যয় করে, আর সত্যিকার অর্থে reversible decisions পরে দ্রুত নেওয়ার জন্য রেখে দেয়।

### ৩. কোনো common vocabulary নেই, তাই কোনো common plan নেই
**আপনি যা দেখেন:** একজন engineer বলেন "আমরা শুধু এটা cache করব," আরেকজন বলেন "আমাদের একটি queue দরকার," তৃতীয়জন বলেন "database-টা shard করো," এবং team দুই সপ্তাহ ধরে তর্ক করে কিছুই সমাধান না করে, কারণ কেউই আসল bottleneck সংজ্ঞায়িত করেনি।

**কেন এটা ঘটে:** common concepts — throughput, latency percentiles, consistency, availability — ছাড়া, architectural তর্কগুলো একে অপরের বিরুদ্ধে opinion ছাড়া আর কিছু নয়। *ভুল* হওয়ার কোনো উপায় নেই, তাই একমত হওয়ারও কোনো উপায় নেই।

**System design কীভাবে এটি সমাধান করে:** এটা vocabulary এবং measuring stick সরবরাহ করে। "আমাদের p99 read latency ৮০০ ms এবং target হলো ২০০ ms; bottleneck হলো uncached fan-out query" — এটা এমন একটি statement যার উপর team কাজ করতে পারে।

### ৪. Interviews যা LeetCode দিয়ে পাস করা যায় না
**আপনি যা দেখেন:** একজন শক্তিশালী coder "Twitter design করো" জিজ্ঞাসা করলে থমকে যান। সন্তুষ্ট করার মতো কোনো test case নেই, একটিমাত্র সঠিক উত্তর নেই, এবং interviewer বারবার "কেন?" জিজ্ঞাসা করতে থাকেন।

**কেন এটা ঘটে:** Algorithm practice আপনাকে *একটি* উত্তর খুঁজে বের করতে প্রশিক্ষণ দেয়। System design-এর কোনো একক উত্তর নেই — এটা আপনাকে বলা constraints-এর অধীনে trade-offs-এর মধ্য দিয়ে চলতে এবং আপনার choices defend করতে বলে। এটা একটি ভিন্ন দক্ষতা, এবং এটাই senior roles-এ প্রকৃতপক্ষে যা hire করা হয়।

**System design কীভাবে এটি সমাধান করে:** এটা আপনাকে একটি repeatable process দেয় — requirements স্পষ্ট করো, scale estimate করো, high-level design sketch করো, একটি component-এ deep-dive করো, trade-offs নাম করো — যা একটি open-ended প্রশ্নকে একটি structured কথোপকথনে রূপান্তরিত করে।

## আপনাকে যে মূল্য দিতে হয়

System design বিনামূল্যে নয়, এবং এটি অতিরিক্ত প্রয়োগ করাও নিজেই একটি failure mode:

- **Analysis paralysis।** এমন একটি product-এর জন্য কয়েক সপ্তাহ ধরে architecture diagrams যার এখনো কোনো user নেই। একটি prototype-এর জন্য code দরকার, capacity model নয়।
- **অকাল complexity।** ১০০ জন daily active user-এর জন্য microservices, Kafka, এবং multi-region failover আপনাকে কোনো সুবিধা ছাড়াই operational কষ্ট এনে দেয়।
- **এমন design যা knowledge-কে ছাড়িয়ে যায়।** সেরা design decisions-এর জন্য প্রকৃত usage data দরকার। কল্পিত traffic patterns-এর জন্য design করা প্রায়ই সহজভাবে design করে measure করার চেয়ে খারাপ।

দক্ষতা হলো জানা যে একটি নির্দিষ্ট সমস্যা কতটা design প্রাপ্য।

## কখন এটা দরকার — এবং কখন নয়

| Design-এ invest করুন যখন | এগিয়ে গিয়ে build করুন যখন |
|---|---|
| Decision-টি পাল্টানো ব্যয়বহুল (data model, service boundaries) | Decision-টি একটি config change যা আপনি পরের সপ্তাহে revisit করতে পারেন |
| একাধিক team বা service ফলাফলের উপর নির্ভরশীল | একজন ব্যক্তিই পুরো surface-এর মালিক |
| Scale, availability, বা correctness targets স্পষ্ট | আপনি যাচাই করছেন কেউ আদৌ এই feature চায় কিনা |
| Failure-এর প্রকৃত cost আছে (money, data loss, safety) | এটা পাঁচজন user-এর একটি internal tool |

## Interviews-এ এটা কেন দেখা যায়

System design rounds বিদ্যমান কারণ senior engineering মূলত judgment, syntax নয়। একজন interviewer যাচাই করছেন: আপনি কি একটি অস্পষ্ট prompt থেকে requirements বের করতে পারেন, আপনি কি estimate করতে পারেন, standard building blocks কী করে তা আপনি জানেন কিনা, এবং — সবচেয়ে গুরুত্বপূর্ণ — আপনি কি articulate করতে পারেন *কেন* আপনি একটির চেয়ে অন্যটি বেছে নিয়েছেন? "কেন"-টাই পুরো evaluation।

## এটা কীভাবে সংযুক্ত

এটাই সেই map যা এরপর যা কিছু আসবে তার জন্য। বাকি course building blocks (load balancers, caches, queues, shards) এবং সেই শক্তিগুলো (scale, latency, failure) পূরণ করবে যা আপনাকে সেগুলোর দিকে হাত বাড়াতে বাধ্য করে। এখান থেকে শুরু করুন যাতে প্রতিটি পরবর্তী topic-এর একটি জায়গা থাকে: আপনি trivia শিখছেন না, আপনি শিখছেন একটি নির্দিষ্ট সমস্যা দেখা দিলে কী ব্যবহার করতে হবে।

**পরবর্তী:** [Functional vs Non-Functional Requirements](../02-functional-vs-non-functional-requirements/why.md) — কারণ যেকোনো design-এর প্রথম ধাপ হলো জানা আপনাকে আসলে কী তৈরি করতে বলা হচ্ছে।
