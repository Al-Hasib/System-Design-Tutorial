# এই বিষয়টি কেন গুরুত্বপূর্ণ: Client-Server Architecture & How the Internet Works

> **এক বাক্যে:** আপনি যে কোনো performance সমস্যা, outage, এবং security ফাঁকফোকর কখনো debug করবেন, তা একটি user-এর browser থেকে আপনার server পর্যন্ত পথের কোনো না কোনো জায়গায় ঘটে থাকে — আর আপনি এমন একটি path debug করতে পারবেন না যা আপনি বর্ণনাই করতে পারেন না।

## এই ধারণার আগের পৃথিবী

যে developer শুধু code থেকে একটি API call করেছে, তার কাছে একটি request একটি একক, অবিভাজ্য বিষয়: আপনি call করেন, আপনি JSON ফেরত পান। বাস্তবে সেই "একটি বিষয়" আসলে একটি DNS lookup, একটি TCP handshake, একটি TLS negotiation, একটি HTTP request, সম্ভবত একাধিক proxy hop, এবং একটি response — যার প্রতিটির নিজস্ব latency, নিজস্ব failure mode, এবং নিজস্ব cache আছে।

যখন app ধীর হয়ে যায়, "API ধীর" একটি diagnosis নয়। ভুল configure করা resolver-এর কারণে DNS resolution দুই সেকেন্ড সময় নিচ্ছে হতে পারে। connection পুনরায় ব্যবহার না হওয়ার কারণে TLS handshake হতে পারে। একটি অনুপস্থিত CDN হতে পারে। path-এর একটি mental model ছাড়া, প্রতিটি অনুসন্ধান শূন্য থেকে শুরু হয়।

## এটি যেসব সমস্যার সমাধান করে

### ১. "site down" যা আসলে site-এর সমস্যা নয়
**আপনি যা দেখেন:** User-রা রিপোর্ট করে site অপ্রাপ্য। Server সুস্থ, CPU স্থির, log-এ কোনো traffic আসার প্রমাণই নেই।

**কেন এটি ঘটে:** সমস্যাটি আপনার code-এর ঊর্ধ্বে — একটি মেয়াদোত্তীর্ণ domain, একটি বাতিল করা IP-কে নির্দেশ করা DNS record, migration-এর পরে propagation delay, একটি firewall rule, মধ্যরাতে মেয়াদোত্তীর্ণ হওয়া একটি certificate। আপনার application request-টি দেখতেই পায়নি।

**এই জ্ঞান কীভাবে এটি সমাধান করে:** request path জানা থাকলে আপনার একটি checklist তৈরি হয়ে যায় যা ধরে ধরে যাচাই করা যায়: domain কি resolve হচ্ছে? এটি কি সঠিক IP-তে resolve হচ্ছে? TCP কি 443-এ connect করছে? TLS certificate কি validate হচ্ছে? প্রতিটি ধাপ অনুমান করার বদলে একটি layer-কে আলাদা করে যাচাই করে।

### ২. Application metrics থেকে যে latency ব্যাখ্যা করা যায় না
**আপনি যা দেখেন:** Server-side timer বলছে handler ২০ ms-এ সম্পন্ন হয়েছে, আর user-রা বলছে page-টি লোড হতে তিন সেকেন্ড লাগছে।

**কেন এটি ঘটে:** application timer শুধু application-এর ভেতরের অংশটুকু মাপে। Round-trip time, DNS lookup, connection setup, TLS negotiation, এবং payload transfer — এই সবকিছু এর বাইরে ঘটে — আর ভৌগোলিকভাবে দূরে থাকা mobile user-এর জন্য এগুলো মোট সময়ে অনেকগুণ বেশি প্রভাব ফেলতে পারে।

**এই জ্ঞান কীভাবে এটি সমাধান করে:** এটি বোঝা যে একটি মহাদেশ-পার round trip-এর একটি কঠোর physical floor প্রায় ১০০-১৫০ ms, তা তাৎক্ষণিকভাবে বুঝিয়ে দেয় যে দশটি sequential request সম্বলিত "chatty" design কখনোই দ্রুত মনে হবে না, আপনি handler যতই optimize করুন না কেন। এটাই CDN, connection পুনর্ব্যবহার, এবং request batching-কে রহস্যময় না রেখে সুস্পষ্ট করে তোলে।

### ৩. দ্বিতীয় server যোগ করা মাত্রই ভেঙে পড়া statefulness
**আপনি যা দেখেন:** app একটি server-এ ঠিকঠাক কাজ করে। আপনি একটি load balancer-এর পেছনে দ্বিতীয় server যোগ করলেন এবং user-রা এলোমেলোভাবে log out হয়ে যেতে শুরু করে।

**কেন এটি ঘটে:** HTTP স্বভাবতই stateless — প্রতিটি request স্বাধীন এবং আগের request-এর কোনো স্মৃতি বহন করে না। আপনি যদি session state একটি server-এর memory-তে জমা রাখেন, তাহলে দ্বিতীয় server-এর কোনো ধারণাই নেই user-টি কে।

**এই জ্ঞান কীভাবে এটি সমাধান করে:** statelessness-কে protocol-এর একটি ইচ্ছাকৃত বৈশিষ্ট্য হিসেবে বোঝা সমাধানটিকে সুস্পষ্ট করে তোলে: state-কে একটি shared store-এ (একটি database বা cache) অথবা request-এর ভেতরেই (একটি signed token) সরিয়ে নিন। এই একটিমাত্র অন্তর্দৃষ্টিই horizontal scaling-এর পূর্বশর্ত।

### ৪. আন্দাজের ওপর ভিত্তি করে নেওয়া security সিদ্ধান্ত
**আপনি যা দেখেন:** "আমরা HTTPS ব্যবহার করি তাই আমরা secure।" অথচ credential URL query string-এ পাঠানো হচ্ছে, যা access log, referrer header, এবং browser history-তে জমা হচ্ছে।

**কেন এটি ঘটে:** প্রতিটি layer আসলে কী রক্ষা করে তা না জেনে, security হয়ে ওঠে cargo-culted। TLS transport-কে encrypt করে — এটি transport-করা data-এর *ভেতরে* আপনি কী রাখছেন, আপনি কাকে authenticate করছেন, বা intermediary-রা কী log করছে, সে সম্পর্কে কিছুই বলে না।

**এই জ্ঞান কীভাবে এটি সমাধান করে:** path-এর একটি স্পষ্ট model আপনাকে ঠিক বলে দেয় প্রতিটি control ঠিক কী কভার করে এবং, গুরুত্বপূর্ণভাবে, কী কভার করে না।

## আপনি যে মূল্য দিচ্ছেন

খুব সামান্যই — এটি একটি foundational knowledge, trade-off-সহ কোনো design choice নয়। একমাত্র প্রকৃত মূল্য হলো সময়: networking-এর অভ্যন্তরীণ বিষয় (congestion control, TCP window scaling, DNS record type-এর পুরো ভাণ্ডার) একটি গভীর কূপ, এবং বেশিরভাগ application engineer-এর কোনো একটি layer-এর খুঁটিনাটির চেয়ে path-এর *আকার*-টাই অনেক বেশি প্রয়োজন। প্রথমে মানচিত্রটি শিখুন; নির্দিষ্ট কোনো সমস্যা দাবি করলে তখন গভীরে যান।

## কখন এটি প্রয়োজন — এবং কখন নয়

| এটি প্রয়োজন যখন | বিস্তারিত পরে করা যায় যখন |
|---|---|
| Latency বা connectivity সমস্যা debug করার সময় | একটি একক service-এর ভেতরে বিশুদ্ধ business logic লেখার সময় |
| ভৌগোলিকভাবে বিতরণকৃত (distributed) যেকোনো কিছু design করার সময় | System একটি local batch job যেখানে কোনো network নেই |
| HTTP, WebSocket, এবং gRPC-এর মধ্যে বেছে নেওয়ার সময় | — |
| Caching, proxy, বা CDN নিয়ে চিন্তা করার সময় | — |

## Interview-এ এটি কেন উঠে আসে

"আপনি যখন একটি browser-এ URL টাইপ করে Enter চাপেন, তখন কী ঘটে?" — এটি এই industry-তে সবচেয়ে বেশি জিজ্ঞাসিত interview প্রশ্নগুলোর একটি, ঠিক এই কারণেই যে এটি একটি মাত্র উত্তরেই একজন candidate-এর mental model-এর গভীরতা প্রকাশ করে দেয়। একটি অগভীর উত্তর "server HTML ফেরত দেয়"-তেই থেমে যায়। একটি শক্তিশালী উত্তর DNS থেকে TCP থেকে TLS থেকে HTTP থেকে response পর্যন্ত হেঁটে যায়, এবং জানে পথে কোথায় cache ও proxy বসে থাকে।

## এটি কীভাবে সংযুক্ত

এটি পুরো course-এর জন্য physical substrate। Load balancer এবং reverse proxy এই path-এর ওপরই বসে। CDN এটিকে সংক্ষিপ্ত করে। Cache এটিকে short-circuit করে। WebSocket এর আকার বদলে দেয়। আর statelessness — এখানে প্রতিষ্ঠিত বৈশিষ্ট্য — horizontal scaling-কে আদৌ সম্ভব করে তোলে।

**পরবর্তী:** [Scalability Basics: Vertical vs Horizontal Scaling](../04-scalability-basics-vertical-vs-horizontal-scaling/why.md) — এই path-এ একটি server যথেষ্ট না হলে কী করতে হবে।
