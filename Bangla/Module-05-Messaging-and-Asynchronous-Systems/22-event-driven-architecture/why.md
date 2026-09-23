# এই বিষয়টি কেন গুরুত্বপূর্ণ: Event-Driven Architecture

> **এক বাক্যে:** Request-driven সিস্টেম জিজ্ঞাসা করে "আমাকে কাকে call করতে হবে?"; event-driven সিস্টেম ঘোষণা করে "এটা ঘটেছে" — এবং এই উল্টে যাওয়াটাই একটি সিস্টেমকে প্রতিটি পরিবর্তন প্রতিটি service-এ ছড়িয়ে না পড়েই বাড়তে দেয়।

## এই ধারণার আগের জগৎ

একটি request-driven সিস্টেমে, business process গুলো call chain হিসেবে এনকোড করা থাকে। Checkout payment-কে call করে, যা fraud-কে call করে, যা risk service-কে call করে, যা ledger-কে call করে। chain টি synchronous, তাই total latency হলো প্রতিটি hop-এর যোগফল এবং total availability হলো প্রতিটি service-এর availability-র গুণফল। প্রতিটি ৯৯.৯% হারে পাঁচটি services আপনাকে দেয় ৯৯.৫% — শুধু arithmetic থেকেই মাসে ৩.৬ ঘণ্টার downtime।

আর business process টি কোথাও থাকে না। "একটি order প্লেস হলে কী ঘটে" তা বর্ণনা করার মতো কোনো একক জায়গা নেই — এটি একটি call graph জুড়ে ছড়িয়ে থাকে যা আপনি শুধুমাত্র এক ডজন repository পড়ে পুনর্গঠন করতে পারবেন।

## এটি যেসব সমস্যার সমাধান করে

### ১. প্রতিটি dependency-র সাথে অবনতিশীল availability
**আপনি যা দেখেন:** আপনার service ৯৯.৯৯% সময় up থাকে এবং আপনার ব্যবহারকারীরা তার চেয়ে অনেক কম অভিজ্ঞতা পায়, কারণ আপনি ততটাই available যতটা আপনি synchronously call করা সবকিছু।

**কেন এটি ঘটে:** Synchronous chains failure probability গুণ করে এবং latency যোগ করে।

**EDA কীভাবে এটি সমাধান করে:** Services asynchronously events publish এবং consume করে। একটি downstream service down থাকার অর্থ হলো তার events জমা হয়, upstream operation ব্যর্থ হয় না। Availability একটি গুণফল হওয়া বন্ধ করে এবং প্রতি-service ভিত্তিক হয়ে ওঠে।

### ২. নতুন capability যোগ করতে বিদ্যমান কোড পরিবর্তন করতে হয়
**আপনি যা দেখেন:** প্রতিটি নতুন feature core services স্পর্শ করে, তাই core services সবসময় দ্বন্দ্বে থাকে, সবসময় সেই দলগুলো দ্বারা review হয় যারা সেই feature-এর মালিক নয়।

**কেন এটি ঘটে:** Orchestration logic callers-এ কেন্দ্রীভূত হয়।

**EDA কীভাবে এটি সমাধান করে:** নতুন আচরণ হলো বিদ্যমান events-এর একটি নতুন consumer। fraud scoring, একটি loyalty program, বা একটি data pipeline যোগ করতে order service-এ শূন্য পরিবর্তন প্রয়োজন। যেসব সিস্টেম *পরিবর্তনের* বদলে *সংযোজনের* মাধ্যমে বৃদ্ধি পায় সেগুলো অনেক বেশি সময় ধরে নিয়ন্ত্রণযোগ্য থাকে।

### ৩. প্রকৃতপক্ষে কী ঘটেছে তার কোনো রেকর্ড নেই
**আপনি যা দেখেন:** একজন গ্রাহক একটি চার্জ নিয়ে বিরোধ তৈরি করে এবং আপনি শুধুমাত্র বর্তমান state পুনর্গঠন করতে পারেন, সেই decision-গুলোর sequence নয় যা এটি তৈরি করেছে। গত মঙ্গলবার একটি bug data নষ্ট করেছে এবং আপনি বলতে পারেন না সঠিক মান কী হওয়া উচিত ছিল।

**কেন এটি ঘটে:** State-mutating সিস্টেম history overwrite করে। Database রেকর্ড করে *কী আছে*, কখনো *কী ঘটেছে* নয়।

**EDA (এবং event sourcing) কীভাবে এটি সমাধান করে:** Event log হলো প্রতিটি fact-এর একটি immutable, ordered রেকর্ড। আপনি audit করতে পারেন, replay করে debug করতে পারেন, একটি corrupted read model শূন্য থেকে পুনর্গঠন করতে পারেন, এবং এমন প্রশ্নের উত্তর দিতে পারেন যা schema ডিজাইন করার সময় কেউ ভাবেনি। Regulated domain-এর জন্য এটি একাই পুরো architecture-এর মূল্য বহন করে।

### ৪. নতুন consumers-এর historical data প্রয়োজন
**আপনি যা দেখেন:** আপনি একটি recommendation engine তৈরি করেন এবং এটিকে দুই বছরের আচরণের উপর train করতে চান যা কখনো ব্যবহারযোগ্য কোনো form-এ সংরক্ষিত ছিল না।

**কেন এটি ঘটে:** Request-driven সিস্টেম কী ঘটেছে তার stream ধরে রাখে না, শুধু ফলাফল রাখে।

**EDA কীভাবে এটি সমাধান করে:** একটি durable log একটি একেবারে নতুন consumer-কে offset শূন্য থেকে শুরু করতে দেয় এবং নিজস্ব view তৈরি করতে সমস্ত history process করতে দেয়। সেই ক্ষমতা — অতীত থেকে একটি নতুন service bootstrap করা — অন্য কোনোভাবে পাওয়া সত্যিই কঠিন।

## যে মূল্য আপনাকে দিতে হবে

Event-driven architecture একটি গুরুতর প্রতিশ্রুতি, এবং এটি হালকাভাবে গ্রহণ করলে খারাপভাবে ব্যর্থ হয়:

- **কোনো global "এখন" নেই।** যেকোনো মুহূর্তে বিভিন্ন services-এর ভিন্ন দৃষ্টিভঙ্গি থাকে। এর কিছু ব্যবহারকারীদের কাছে অদৃশ্য; এর কিছু একটি correctness সমস্যা যা আপনাকে স্পষ্টভাবে ঘিরে ডিজাইন করতে হবে।
- **Control flow অদৃশ্য।** আপনি একটি function পড়ে জানতে পারবেন না এরপর কী ঘটবে। একটি business process বোঝার অর্থ হলো পুরো সিস্টেম জুড়ে subscriptions বোঝা — যারা এটি গ্রহণ করেছে তাদের কাছ থেকে আসা সবচেয়ে বড় অভিযোগ।
- **Debugging-এর জন্য infrastructure প্রয়োজন।** correlation IDs, distributed tracing, এবং কেন্দ্রীভূত logs ছাড়া, আটটি asynchronous hop জুড়ে বিস্তৃত একটি failure নির্ণয় করা প্রায় আশাহীন। observability আগে তৈরি করুন, পরে নয়।
- **Event schemas স্থায়ী public contracts।** একবার প্রকাশিত এবং অজানা পক্ষের দ্বারা consume হলে, একটি event-এর আকার পরিবর্তন করা খুব কঠিন। প্রথম দিন থেকেই versioning discipline প্রয়োজন।
- **Duplicates এবং reordering নিশ্চিত।** প্রতিটি consumer-কে idempotent হতে হবে এবং সাধারণত out-of-order আগমন সহ্য করতে হবে।
- **এটি ছোট সিস্টেমের জন্য ভুল default।** একটি পাঁচজনের team একটি CRUD product তৈরি করলে একটি monolith-এর ভেতরে synchronous calls দিয়ে দ্রুত এগোবে এবং শান্তিতে ঘুমাবে। EDA-এর খরচ organizational scale-এ ন্যায্যতা পায়, code scale-এ নয়।

## কখন আপনার এটি প্রয়োজন — এবং কখন নেই

| Event-driven-এ যান যখন | Request-driven থাকুন যখন |
|---|---|
| অনেক স্বাধীন team সমান্তরালে বিকশিত হতে হয় | একটি ছোট team সবকিছুর মালিক |
| একটি action-এর প্রতিক্রিয়া ক্রমাগত বাড়তে থাকে | Workflow সংক্ষিপ্ত, নির্দিষ্ট, এবং তাৎক্ষণিক উত্তর প্রয়োজন |
| Audit trails বা replay প্রকৃত প্রয়োজনীয়তা | পুরো operation জুড়ে strong consistency প্রয়োজন |
| আপনাকে burst buffer করতে এবং availability decouple করতে হবে | Decoupling-এর চেয়ে debugging simplicity বেশি গুরুত্বপূর্ণ |
| Consumers-এর processing rate খুবই ভিন্ন | আপনার tracing এবং monitoring maturity নেই |

## এটি কেন interview-এ দেখা যায়

EDA অনেক প্রতিক্রিয়াশীল subsystem সহ ডিজাইনে দেখা যায়: e-commerce checkout, ride-sharing state transitions, notification platforms, এবং analytics pipelines। Interviewer-রা দেখতে চায় আপনি কি স্পষ্ট করে বলতে পারেন আপনি কী *পান* (decoupling, স্বাধীন scaling, auditability, resilience) এবং — আরও তাৎপর্যপূর্ণভাবে — আপনি কী *হারান* (তাৎক্ষণিক consistency, traceability, সহজ debugging)। যে candidate EDA প্রস্তাব করে এবং তারপর বলে "কিন্তু আমি payment authorization synchronous রাখব, কারণ order নিশ্চিত করার আগে ব্যবহারকারীর একটি নির্দিষ্ট হ্যাঁ বা না প্রয়োজন" সে ঠিক সেই judgment দেখাচ্ছে যা যাচাই করা হচ্ছে।

## এটি কীভাবে সংযুক্ত

EDA তৈরি হয়েছে **message queues** (topic 20)-এর উপর **Pub/Sub** (topic 21) দিয়ে, সাধারণত replay-এর জন্য Kafka-এর মতো একটি log-based queue। এটিই সেই communication backbone যা **microservices**-কে সত্যিকারের স্বাধীন করে তোলে (topics 30-31), এটি **idempotency** এবং **eventual consistency**-এর উপর নির্ভরশীল (topic 29), এবং **Saga pattern**-এর মাধ্যমে cross-service workflows বাস্তবায়ন করে (topic 28)। এর events **stream processing**-কে খাদ্য যোগায় (topic 23), এবং এটি **observability**-কে (topic 43) একটি কঠিন পূর্বশর্ত করে তোলে। **Domain-driven design** (topic 32) হলো যেখান থেকে ভালো event boundaries আসে।

**পরবর্তী:** [Batch vs Stream Processing](../23-batch-vs-stream-processing/why.md) — আপনি এখন যে events তৈরি করছেন তা প্রকৃতপক্ষে process করার দুটি উপায়।
</content>
