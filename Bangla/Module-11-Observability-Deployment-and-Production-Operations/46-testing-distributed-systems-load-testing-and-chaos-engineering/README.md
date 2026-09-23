# Testing Distributed Systems: Load Testing & Chaos Engineering

**Difficulty:** Advanced

## Learning Objectives

- ব্যাখ্যা করুন কেন unit এবং integration test আপনাকে বলে না যে কোনো সিস্টেম বাস্তব production load বা বাস্তব infrastructure failure-এ টিকে থাকতে পারবে কিনা।
- load testing, stress testing, এবং soak testing বর্ণনা করুন, এবং প্রতিটি ঠিক কোন প্রশ্নের উত্তর দেয় তা ব্যাখ্যা করুন।
- chaos engineering-এর দর্শন ব্যাখ্যা করুন: ইচ্ছাকৃতভাবে failure inject করে confidence তৈরি করা, এটা প্রমাণ করার জন্য নয় যে কিছু কখনো ভাঙবে না।
- "game day" চালানোর প্র্যাকটিস এবং chaos experiment নিরাপদে চালানোর জন্য প্রয়োজনীয় blast-radius discipline বর্ণনা করুন।
- পরিচিত failure-handling mechanism (retries, circuit breakers, redundancy) থাকা একটি সিস্টেমের জন্য একটি বেসিক resilience-testing পরিকল্পনা ডিজাইন করুন।

## Script

### Hook / Intro

কোর্সের এই পর্যায়ে এসে আমরা retries, circuit breakers, replication, redundancy, এবং failover (Module 6-এর resilience patterns) দিয়ে সিস্টেম ডিজাইন করে ফেলেছি। এখন একটা অস্বস্তিকর প্রশ্ন—যেটা প্রায় কেউই জিজ্ঞেস করে না, যতক্ষণ না একটা প্রকৃত incident সেটা করতে বাধ্য করে: আপনি কীভাবে *জানবেন* যে ওগুলোর কোনোটা আসলে কাজ করে? একটা unit test যাচাই করতে পারে যে আপনার circuit breaker-এর state machine নিজে থেকে ঠিকভাবে transition করছে কিনা। কিন্তু এটা আপনাকে বলতে পারে না যে আপনার circuit breaker-এর timeout আসলেই আপনার real network-এর latency distribution-এর জন্য ভালোভাবে tune করা কিনা, অথবা real production traffic pattern-এর অধীনে একটি real downstream dependency সত্যিই স্লো হয়ে গেলে আপনার service আসলে গ্রেসফুলি degrade করে কিনা। আজ আমরা এই দুটি discipline নিয়ে আলোচনা করব যা এই ফাঁকটা পূরণ করে: load testing, যা বলে দেয় বাস্তবসম্মত (বা extreme) traffic-এর অধীনে আপনার সিস্টেম কেমন আচরণ করে, এবং chaos engineering, যা বলে দেয় বাস্তবসম্মত (বা extreme) failure-এর অধীনে এটা কেমন আচরণ করে—ইচ্ছাকৃতভাবে, নিজের সময়সূচি অনুযায়ী, একটা প্রকৃত outage-এর সময় প্রথমবার জানার বদলে।

### Load Testing: এটা কি বাস্তব Traffic টিকিয়ে রাখতে পারে?

**Load testing** মানে হলো আপনার সিস্টেমের বিরুদ্ধে synthetic traffic তৈরি করা এবং এটা কেমন আচরণ করে তা পর্যবেক্ষণ করা—কিন্তু "load testing" আসলে বেশ কয়েকটি ভিন্ন প্রশ্নকে একসাথে ধরে রাখা একটি umbrella term। **Load testing** (সংকীর্ণ অর্থে) প্রত্যাশিত peak traffic-এ আচরণ যাচাই করে—সিস্টেম কি এই Black Friday-তে আপনার প্রত্যাশিত সর্বোচ্চ load-এ টিকে থাকতে পারবে? **Stress testing** প্রত্যাশিত peak-এর অনেক বেশি traffic পাঠায়, বিশেষভাবে breaking point খুঁজে বের করার জন্য এবং—ঠিক সমান গুরুত্বপূর্ণভাবে—সিস্টেম *কীভাবে* ব্যর্থ হয় তা পর্যবেক্ষণ করার জন্য: এটা কি গ্রেসফুলি degrade করে (কম-priority কাজ ঝেড়ে ফেলে, পরিষ্কারভাবে 429 রিটার্ন করে, Module 6-এর rate limiting অনুযায়ী) নাকি এটা catastrophic-ভাবে ব্যর্থ হয় (cascading failure যা সম্পর্কহীন service-গুলোকেও ফেলে দেয়, Module 6-এর circuit breaker আলোচনা অনুযায়ী)? **Soak testing** (বা endurance testing) দীর্ঘ সময়ের জন্য—ঘণ্টার পর ঘণ্টা বা দিনের পর দিন—একটি sustained, moderate load চালায়, বিশেষভাবে সেই সমস্যাগুলো ধরার জন্য যেগুলো কেবল সময়ের সাথে সাথে প্রকাশ পায়: memory leak, ধীরে ধীরে resource exhaustion (connection pool ধীরে ধীরে leak হওয়া, log দিয়ে disk ভরে যাওয়া), অথবা এমন degradation যা একটা ছোট test চালানোর সময় ধরাই পড়বে না। k6, Locust, এবং Gatling-এর মতো টুল বড় স্কেলে এই synthetic load তৈরি করার জন্য সাধারণভাবে ব্যবহৃত হয়, এবং "আপনার ব্যবহারকারীদের আগে নিজেই এটা টেস্ট করুন" এই discipline-টাই নির্ধারণ করে দেয় আপনি কোনো মঙ্গলবার বিকেলে আপনার সিস্টেমের প্রকৃত breaking point খুঁজে পাবেন, নাকি সেটা একটা বাস্তব, revenue-impacting traffic spike-এর সময় আবিষ্কার করবেন।

### Chaos Engineering: এটা কি বাস্তব Failure টিকিয়ে রাখতে পারে?

Load testing "বেশি traffic-এর অধীনে কী হয়" এই প্রশ্নের উত্তর দেয়। **Chaos engineering** একটি ভিন্ন প্রশ্নের উত্তর দেয়: "যখন আমার infrastructure-এর একটা অংশ সত্যিই ভেঙে যায়, এখনই, production বা production-এর মতো পরিবেশে, তখন কী হয়?" এর মূল দর্শন, যা প্রকাশ্যে প্রথম চালু করেছিল Netflix-এর Chaos Monkey, শুরুতে সত্যিই counter-intuitive মনে হয়: ইচ্ছাকৃতভাবে এবং random-ভাবে failure ঘটানো—একটা server instance মেরে ফেলা, network latency যোগ করা, একটা dependency timeout সিমুলেট করা—যাতে *confidence* তৈরি হয় যে আপনার সিস্টেমের failure-handling mechanism (Module 6-এর সেই সব: retries, circuit breakers, redundancy, failover) সত্যিই ডিজাইন অনুযায়ী কাজ করে, এমন পরিস্থিতিতে যা আপনি পুরোপুরি আগে থেকে script করতে পারবেন না। যুক্তিটা হলো: আপনার সিস্টেম একসময় বাস্তব failure মুখোমুখি হবেই, আপনি সময় বেছে নিন বা না নিন। Chaos engineering কেবল জোর দেয় যে সময়টা বেছে নেওয়া হোক—আদর্শভাবে business hours-এ, যখন team dashboard দেখছে এবং হস্তক্ষেপ করার জন্য প্রস্তুত, বরং রাত ৩টায় একটা প্রকৃত incident-এর সময় যখন একটা real customer impact clock চলছে তখন নয়।

এটা "randomly production ভেঙে দেখা কী হয়" ধরনের বেপরোয়ামি নয়—আসল practice হলো disciplined এবং incremental। আপনি শুরু করেন একটা **hypothesis** দিয়ে: "যদি payments service-এর primary database replica ব্যর্থ হয়, তাহলে সামান্য একটা latency blip ছাড়া traffic ১০ সেকেন্ডের মধ্যে secondary-তে failover করা উচিত।" আপনি একটা টাইট-বাউন্ড **blast radius** নির্ধারণ করেন: প্রথমে traffic-এর একটা ছোট শতাংশ বা একটা non-critical environment-এ experiment চালান, সাথে থাকে একটা automatic "abort" trigger যদি key metric (আপনার observability সেটআপের সেই মেট্রিকগুলো) একটা বিপদসীমা অতিক্রম করে। আপনি experiment চালান, দেখেন প্রকৃত আচরণ hypothesis-এর সাথে মিলল কিনা, আর—এটাই আসল উদ্দেশ্য—প্রায় সবসময়ই এমন কিছু খুঁজে পাবেন যা আপনার ডিজাইন ধরে নিয়েছিল যে কাজ করবে কিন্তু আসলে কোনো নির্দিষ্ট, আগে-অদৃশ্য উপায়ে ঠিকমতো করে না। এরপর আপনি সেটা ঠিক করেন, এবং confidence বাড়ার সাথে সাথে ধীরে ধীরে blast radius এবং experiment-এর জটিলতা বাড়ান।

### Game Days

অনেক প্রতিষ্ঠান এটাকে একটা **game day**-তে formalize করে: একটা পরিকল্পিত, নির্ধারিত exercise যেখানে একটা team ইচ্ছাকৃতভাবে একটা নির্দিষ্ট failure scenario সিমুলেট করে—একটা region ডাউন হয়ে যাওয়া, একটা critical dependency অনুপলব্ধ হয়ে যাওয়া, একটা data corruption event—এবং প্রকৃত incident response প্র্যাকটিস করে, শুধু সিস্টেমের automated recovery নয়। এটা এমন কিছু টেস্ট করে যা শুধু automated chaos experiment একা করতে পারে না: *মানুষ* এবং *runbook* (একটা পরিচিত failure mode-এর প্রতিক্রিয়ার জন্য documented procedure) আসলেই কাজ করে কিনা তা সিমুলেটেড চাপের মধ্যে, on-call engineer সময়মতো সঠিক dashboard খুঁজে পেতে পারে কিনা, এবং escalation process যেভাবে documented আছে সেভাবেই কাজ করে কিনা। একটা perfect automated failover থাকা সিস্টেমও প্রকৃত incident-এ পরিণত হতে পারে যদি যেসব মানুষদের জড়িত হওয়া দরকার তারা না জানে কী ঘটছে তা কীভাবে diagnose করতে হয়—game day সরাসরি এই ফাঁকটা পূরণ করে।

### Real-World Example

ধরুন redundant database replica এবং একটা automated failover mechanism (Module 3-এর replication) থাকা একটা payments system—ডিজাইন বলছে failover ১০ সেকেন্ডের মধ্যে সম্পন্ন হওয়া উচিত। একটা chaos experiment এটা সরাসরি টেস্ট করে: একটা নির্ধারিত সময়ে, team dashboard দেখার সময়, primary replica ইচ্ছাকৃতভাবে মেরে ফেলা হয়, এবং team দেখে আসলে কী ঘটে—হয়তো failover ডিজাইন অনুযায়ী ৮ সেকেন্ডে সম্পন্ন হয়, অথবা হয়তো দেখা যায় application-এর connection pool ঠিকমতো reconnect করে না এবং একটা পুরো restart প্রয়োজন হয়, যাতে দুই সেকেন্ডের বদলে পুরো দুই মিনিট লাগে—একটা প্রকৃত ফাঁক ডিজাইন করা আচরণ ও প্রকৃত আচরণের মধ্যে, যা একা একটা ডিজাইন ডকুমেন্ট কখনোই সামনে আনতে পারত না। আলাদাভাবে, একটা পরিচিত high-traffic sales event-এর আগে একটা load test গত বছরের peak-এর ৩ গুণ synthetic traffic চালায়, যা প্রকাশ করে যে একটা নির্দিষ্ট downstream inventory service ২.৫ গুণে গিয়েই timeout শুরু করে দেয়—যা এই বছরের প্রত্যাশিত peak-এর মধ্যেই—team-কে এই বাস্তব capacity সমস্যা ঠিক করার জন্য কয়েক সপ্তাহ সময় দেয়, তা সরাসরি event চলাকালীন আবিষ্কার করার বদলে।

### Recap

Unit এবং integration test আলাদা করে logic যাচাই করে; সেগুলো আপনাকে বলে না সিস্টেম বাস্তব production load বা বাস্তব infrastructure failure টিকিয়ে রাখতে পারবে কিনা—তার জন্য প্রয়োজন ইচ্ছাকৃতভাবে দুটোই টেস্ট করা। Load testing (প্রত্যাশিত peak-এ), stress testing (peak-এর অনেক বেশি, breaking point খুঁজে বের করতে এবং failure mode পর্যবেক্ষণ করতে), এবং soak testing (দীর্ঘ সময় ধরে sustained load, ধীর leak এবং ধীরে ধীরে degradation ধরতে) প্রতিটি traffic সম্পর্কিত আলাদা একটা প্রশ্নের উত্তর দেয়। Chaos engineering ইচ্ছাকৃতভাবে বাস্তব failure inject করে—একটা স্পষ্ট hypothesis এবং একটা disciplined, bounded blast radius সহ—যাতে confidence তৈরি হয় যে আপনার resilience mechanism আসলেই কাজ করে, শুধু ডিজাইন ডকুমেন্ট এমনটা বলে বলে ধরে নেওয়ার বদলে। Game day এটাকে আরেক ধাপ এগিয়ে নিয়ে মানুষ এবং runbook টেস্ট করে, শুধু automated system নয়, "failure টিকিয়ে রাখার জন্য ডিজাইন করা" আর "আসলেই failure টিকিয়ে রাখতে পারা, মানুষের প্রতিক্রিয়াসহ" এর মধ্যে শেষ ফাঁকটা পূরণ করে।

### What's Next

আমরা এখন একটা একক সিস্টেমের resilience ইচ্ছাকৃতভাবে টেস্ট করে ফেলেছি। এই module-এর শেষ video একটা লেভেল উপরে যায়: যখন failure একটা server বা একটা dependency নয়, বরং একটা পুরো region, তখন কী হয়—এবং multi-region architecture এবং disaster recovery planning কীভাবে ঠিক করে দেয় সেটা কতটা খারাপ হতে পারে।

## Key Takeaways

- Unit এবং integration test আলাদা করে logic যাচাই করে; সেগুলো প্রমাণ করে না যে সিস্টেম বাস্তব production load বা বাস্তব infrastructure failure টিকিয়ে রাখতে পারবে।
- Load testing প্রত্যাশিত peak traffic-এ আচরণ যাচাই করে; stress testing peak-এর ওপারে গিয়ে breaking point খুঁজে বের করে এবং failure mode পর্যবেক্ষণ করে; soak testing দীর্ঘ সময় ধরে sustained load চালিয়ে ধীর leak এবং ধীরে ধীরে degradation ধরে।
- Chaos engineering ইচ্ছাকৃতভাবে failure inject করে—একটা স্পষ্ট hypothesis এবং একটা disciplined, bounded blast radius সহ—যাতে confidence তৈরি হয় যে resilience mechanism (retries, circuit breakers, failover) বাস্তব পরিস্থিতিতে আসলেই কাজ করে।
- Chaos experiment প্রায় সবসময় ডিজাইন করা আচরণ ও প্রকৃত আচরণের মধ্যে একটা ফাঁক প্রকাশ করে—এটাই মূল উদ্দেশ্য, একটা খারাপভাবে চালানো experiment-এর লক্ষণ নয়।
- Game day মানুষ এবং runbook টেস্ট করে, শুধু automated system নয়—একটা perfect automated failover-ও একটা প্রকৃত incident-এ পরিণত হতে পারে যদি on-call response কাজ না করে।
