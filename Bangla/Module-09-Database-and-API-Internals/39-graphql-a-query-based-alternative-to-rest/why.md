# কেন এই Topic গুরুত্বপূর্ণ: GraphQL — REST-এর একটি Query-ভিত্তিক বিকল্প

> **এক বাক্যে:** REST endpoints-এর shape backend team ঠিক করে, এবং তারপর প্রতিটি client হয় প্রয়োজনের চেয়ে অনেক বেশি download করে, অথবা একটি screen তৈরি করতে ছয়টি request পাঠায় — GraphQL response-এর shape ঠিক করার দায়িত্ব সেই client-এর হাতে দেয় যে আসলে জানে তার কী দরকার।

## এই ধারণার আগের পৃথিবী

একটি mobile app একটি user profile screen render করে: নাম, avatar, follower count, এবং সর্বশেষ তিনটি post-এর title।

REST দিয়ে এটি `GET /users/42` call করে, যা পুরো user object ফেরত দেয় — bio, settings, notification preferences, এবং timestamps সহ পঞ্চাশটি field — কারণ সেই endpoint web app, admin panel, এবং আরও তিনটি screen-কে সার্ভ করে। এরপর এটি `GET /users/42/posts` call করে, এবং প্রতিটি post-এর জন্য একটি count পেতে `GET /posts/{id}/comments` call করে। ছয়টি request, প্রতিটি একটি mobile network-এ একটি সম্পূর্ণ round trip, এবং payload-টি screen-এ যা দেখানো হয় তার চেয়ে কয়েকগুণ বেশি।

mobile team একটি trimmed endpoint চায়। backend team `GET /users/42/profile-summary` যোগ করে। তারপর tablet team সামান্য ভিন্ন একটি চায়। তারপর watch app। ছয় মাস পর প্রায়-অভিন্ন চৌদ্দটি endpoint তৈরি হয়ে যায়, প্রতিটির নিজস্ব tests এবং নিজস্ব maintenance সহ, এবং প্রতিটি UI change-এর জন্য এখনও একটি backend deploy দরকার হয়।

## এটি যেসব সমস্যার সমাধান করে

### ১. Over-fetching এবং Under-fetching
**আপনি যা দেখেন:** অব্যবহৃত fields-এ ভরা payloads, সাথে এমন screens যেগুলো তৈরি করতে একাধিক sequential request দরকার হয়।

**কেন এমন হয়:** একটি REST endpoint একটি fixed resource shape ফেরত দেয়, যা একবার সব consumers-এর জন্য বেছে নেওয়া হয়েছিল। এটি একটি client-এর জন্য সঠিক হতে পারে কিন্তু বাকিদের জন্য ভুল।

**GraphQL কীভাবে এটি সমাধান করে:** Client একটি query পাঠায় যা ঠিক কোন fields এবং relationships দরকার তা বর্ণনা করে, এবং ঠিক সেটাই পায় — একটি request, একটি response, অতিরিক্ত কিছু নয়। high-latency, metered connection-এ থাকা mobile clients-এর জন্য, অতিরিক্ত bytes এবং অতিরিক্ত round trips দুটোই দূর করা একটি substantial, user-visible improvement।

### ২. Frontend পরিবর্তনের জন্য Backend Deploys প্রয়োজন
**আপনি যা দেখেন:** একটি screen-এ একটি field যোগ করা মানে একটি backend ticket, একটি backend deploy, এবং একটি অপেক্ষা।

**কেন এমন হয়:** Response shapes backend code-এ থাকে।

**GraphQL কীভাবে এটি সমাধান করে:** যদি field-টি ইতিমধ্যে schema-তে exist করে, তাহলে frontend শুধু সেটি চেয়ে নেয়। Frontend teams backend coordination ছাড়াই iterate করতে পারে, যা প্রায়ই এটি adopt করার সবচেয়ে বড় organizational কারণ।

### ৩. কখনো শেষ না হওয়া Versioning
**আপনি যা দেখেন:** `/v1`, `/v2`, `/v3` একসাথে চলছে, পুরনো versions অনির্দিষ্টকালের জন্য চালু রাখা হচ্ছে কারণ বাস্তবে থাকা mobile clients-দের upgrade করতে বাধ্য করা যায় না।

**কেন এমন হয়:** একটি shared response shape-এ যেকোনো পরিবর্তন সম্ভাব্যভাবে breaking, তাই একমাত্র নিরাপদ পদক্ষেপ হলো একটি নতুন version।

**GraphQL কীভাবে এটি সমাধান করে:** একটি field যোগ করলে কারও কিছু ভাঙে না, কারণ clients শুধু যা চেয়েছে তাই পায়। Fields সরানোর বদলে deprecate করা হয়, এবং — যেহেতু প্রতিটি query explicit — আপনি ঠিক measure করতে পারেন কোন clients এখনও একটি deprecated field request করছে, সেটি retire করার আগে। সেই measurability REST-এর তুলনায় একটি real advantage, যেখানে কে কোন field পড়ছে তা বোঝার উপায় নেই।

### ৪. Discovery এবং Integration Friction
**আপনি যা দেখেন:** নতুন clients-দের API call করার আগে একটি walkthrough এবং একটি পুরনো wiki page দরকার হয়।

**কেন এমন হয়:** REST APIs conventionally structured, কিন্তু self-describing নয় যতক্ষণ না কেউ একটি OpenAPI spec maintain করে।

**GraphQL কীভাবে এটি সমাধান করে:** Schema strongly typed এবং introspectable, তাই tooling বিনামূল্যে autocompletion, type checking, এবং generated client types দেয়। Contract implementation থেকে drift করতে পারে না, কারণ এটিই implementation।

## যে মূল্য আপনাকে দিতে হয়

GraphQL real সমস্যার সমাধান করে এবং একই সাথে নতুন সমস্যার একটি স্বতন্ত্র সেট তৈরি করে:

- **HTTP caching কাজ করা বন্ধ করে দেয়।** প্রতিটি query সাধারণত একটি single `/graphql` endpoint-এ একটি `POST`, তাই CDNs, reverse proxies, এবং browser caches — যেগুলো URL এবং method-এর উপর key করে — সাহায্য করতে পারে না। এর বদলে আপনাকে field বা resolver level-এ caching তৈরি করতে হবে। read-heavy public content-এর জন্য, এটি সত্যিকারের একটি বড় ক্ষতি।
- **N+1 problem হলো default।** authors সহ 100টি posts-এর একটি query naive ভাবে 1 + 100 database query issue করে, কারণ প্রতিটি field independently resolve হয়। DataLoader-style batching এটি ঠিক করে, এবং আপনাকে এটি ইচ্ছাকৃতভাবে এবং সব জায়গায় apply করতে হবে — এটি automatic নয়।
- **Clients expensive queries লিখতে পারে।** Deeply nested বা wide queries pathologically costly হতে পারে, এবং একটি public API-তে এটি একটি denial-of-service vector। Mitigations (query depth limits, complexity scoring, persisted queries, allowlists) optional নয়, mandatory — এবং প্রতিটিই নতুন machinery যোগ করে।
- **Rate limiting কঠিন।** "প্রতি মিনিটে 100 requests" অর্থহীন যখন একটি request আরেকটির চেয়ে হাজার গুণ বেশি expensive হতে পারে। আপনার cost-based limiting দরকার।
- **Observability কঠিন।** প্রতিটি request একটি 200 status সহ একই endpoint-এ hit করে, তাই per-endpoint dashboards, error rates, এবং latency breakdowns — সবকিছুরই GraphQL-aware instrumentation দরকার।
- একটি client এবং simple প্রয়োজনের একটি API-এর জন্য **এটি অনেক বেশি পরিমাণ infrastructure**, যেখানে REST কম code এবং ভুল হওয়ার কম সুযোগ দেয়।

## কখন এটি আপনার দরকার — এবং কখন নয়

| GraphQL কখন ব্যবহার করবেন | REST কখন ব্যবহার করবেন |
|---|---|
| সত্যিকারের ভিন্ন data প্রয়োজনসহ অনেক client type | একটি client, বা uniform প্রয়োজনসহ clients |
| Data গভীর relationships সহ graph-আকৃতির | Resources flat এবং CRUD-আকৃতির |
| Mobile clients-এর ন্যূনতম payloads এবং কম round trips দরকার | HTTP caching এবং CDN offload গুরুত্বপূর্ণ |
| Frontend teams-এর independently iterate করা দরকার | API public এবং consume করা সহজ হতে হবে |
| আপনি একাধিক backend service aggregate করছেন | File uploads এবং downloads প্রধান |

একটি common এবং sensible মধ্যবর্তী পথ: first-party apps-এর জন্য internal BFF layer হিসেবে GraphQL, এবং public ও partner API-এর জন্য REST।

## কেন এটি Interviews-এ দেখা যায়

API design নিয়ে আলোচনার সময়, বিশেষ করে mobile-heavy বা multi-client systems-এর জন্য, GraphQL একটি বিকল্প হিসেবে দেখা যায়। Interviewers যে signal-টি খোঁজেন তা হলো balance: যারা শুধু এর সুবিধার জন্য এটি প্রস্তাব করে তারা তাদের চেয়ে দুর্বল যারা খরচগুলোও উল্লেখ করে — বিশেষ করে HTTP caching হারানো এবং N+1 resolver problem, কারণ এই দুটোই production-এ team-দের সত্যিকারভাবে কামড় দেয়। এটিকে REST-এর একটি সম্পূর্ণ replacement হিসেবে না দেখিয়ে একটি client-driven BFF হিসেবে frame করা সাধারণত সবচেয়ে পরিণত (mature) position হিসেবে ধরা হয়।

## এটি কীভাবে সংযুক্ত

GraphQL মূলত একটি generalized **BFF** (topic 9), যেখানে aggregation একটি backend team দ্বারা hardcode করার বদলে client দ্বারা query time-এ নির্দিষ্ট করা হয়। এটি **REST over HTTP** (topic 6)-এর একটি বিকল্প, এবং এর caching সমস্যা সেই HTTP semantics ত্যাগ করার একটি সরাসরি ফলাফল যা **caching** (topic 17) এবং **CDNs** (topic 18)-কে কাজ করায়। Resolver batching হলো ঠিক সেই একই N+1 problem যা **microservices communication** (topic 31)-এ দেখা যায়, এবং query-cost limiting হলো **rate limiting** (topic 25)-এর একটি specialized রূপ।

**পরবর্তী:** [Distributed Locking](../../Module-10-Distributed-Coordination-and-Scale-Techniques/40-distributed-locking-redlock-zookeeper-and-etcd/why.md) — machines জুড়ে exclusive access coordinate করা।
