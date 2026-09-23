# Practice & Interview Questions

**১. কেন unit এবং integration test আপনাকে বলে না যে একটা সিস্টেম বাস্তব production load বা বাস্তব infrastructure failure টিকিয়ে রাখতে পারবে কিনা?**
Unit এবং integration test নিয়ন্ত্রিত, অনুমানযোগ্য পরিস্থিতিতে আলাদা করে logic যাচাই করে—উদাহরণস্বরূপ, এগুলো নিশ্চিত করতে পারে একটা circuit breaker-এর state machine ঠিকভাবে transition করে, কিন্তু এটা বলতে পারে না এর timeout বাস্তব network latency-এর জন্য ভালোভাবে tune করা কিনা, অথবা বাস্তব production traffic pattern-এর অধীনে একটা real downstream dependency সত্যিই স্লো হয়ে গেলে সিস্টেম আসলে কেমন আচরণ করে।

**২. Load testing, stress testing, এবং soak testing-এর মধ্যে পার্থক্য করুন।**
Load testing প্রত্যাশিত peak traffic-এ আচরণ যাচাই করে। Stress testing বিশেষভাবে breaking point খুঁজে বের করতে এবং failure mode পর্যবেক্ষণ করতে প্রত্যাশিত peak-এর অনেক বেশি পাঠায়। Soak testing দীর্ঘ সময় ধরে একটা sustained, moderate load চালায় এমন সমস্যা ধরতে (যেমন memory leak বা ধীরে ধীরে resource exhaustion) যেগুলো শুধু সময়ের সাথে সাথে দেখা যায়, একটা ছোট test-এ নয়।

**৩. Stress testing-এ একটা সিস্টেমের "গ্রেসফুলি ব্যর্থ হওয়া" বনাম "catastrophically ব্যর্থ হওয়া" মানে কী, এবং প্রতিটির একটা উদাহরণ দিন।**
গ্রেসফুলি ব্যর্থ হওয়া মানে সিস্টেম নিয়ন্ত্রিতভাবে অতিরিক্ত কাজ ঝেড়ে ফেলে বা প্রত্যাখ্যান করে—যেমন, capacity পার হয়ে গেলে পরিষ্কারভাবে 429 Too Many Requests রিটার্ন করা (Module 6-এর rate limiting মনে করুন)। Catastrophically ব্যর্থ হওয়া মানে overload cascade হয়ে যায়—যেমন, একটা overwhelmed service timeout তৈরি করে যা সম্পর্কহীন service-গুলোকেও ব্যর্থ করে দেয়, circuit breaker এবং bulkhead যে isolation দেওয়ার কথা তার অভাবে।

**৪. Chaos engineering-এর মূল দর্শন কী?**
ইচ্ছাকৃতভাবে এবং সক্রিয়ভাবে একটা সিস্টেমে বাস্তব failure inject করা, নিজের সময়সূচি অনুযায়ী এবং team দেখার সময়, যাতে confidence তৈরি হয় যে resilience mechanism (retries, circuit breakers, redundancy, failover) আসলেই কাজ করে—ডিজাইন ডকুমেন্ট এমনটা বলে বলে ধরে নেওয়ার বদলে, এবং একটা প্রকৃত, অনিয়ন্ত্রিত incident-এর সময় প্রথমবার উল্টোটা জানার বদলে।

**৫. একটা disciplined chaos engineering experiment-এর পাঁচটি (বা ছয়টি) ধাপ বর্ণনা করুন।**
একটা failure-এর অধীনে প্রত্যাশিত আচরণ নিয়ে একটা নির্দিষ্ট hypothesis তৈরি করা, একটা bounded blast radius (ছোট traffic শতাংশ বা non-critical environment) সংজ্ঞায়িত করা সাথে একটা abort trigger, বাস্তব failure inject করে experiment চালানো, প্রকৃত আচরণ hypothesis-এর সাথে মিলল কিনা পর্যবেক্ষণ করা, যা পাওয়া গেছে সেই ফাঁক ঠিক করা, এবং ভবিষ্যৎ experiment-এর blast radius ও জটিলতা ধীরে ধীরে বাড়ানো।

**৬. একটা chaos experiment চালানোর আগে "blast radius" সংজ্ঞায়িত করা কেন গুরুত্বপূর্ণ?**
একটা bounded blast radius ছাড়া, একটা experiment একটা ছোট, নিয়ন্ত্রিত test-এর বদলে সব ব্যবহারকারীকে প্রভাবিত করে একটা প্রকৃত, অনিয়ন্ত্রিত outage ঘটাতে পারে। আগে থেকেই এটা সংজ্ঞায়িত করা (একটা ছোট traffic শতাংশ, একটা non-critical environment, observability metric-এর সাথে যুক্ত একটা automatic abort trigger) chaos engineering-কে একটা disciplined practice করে তোলে, "randomly production ভেঙে ফেলা" ধরনের বেপরোয়া আচরণ নয়।

**৭. একটা chaos experiment কেন "প্রায় সবসময়ই" ডিজাইন করা আচরণ ও প্রকৃত আচরণের মধ্যে একটা ফাঁক প্রকাশ করে, এবং এটাকে exercise-এর ব্যর্থতা না বলে মূল্যবান কেন মনে করা হয়?**
বাস্তব সিস্টেম সূক্ষ্ম আচরণ জমা করে—connection pool-এর অদ্ভুততা, timing সংক্রান্ত ধারণা, configuration drift—যা একটা ডিজাইন ডকুমেন্ট ধরতে পারে না এবং শুধু প্রকৃত failure পরিস্থিতিতেই দেখা যায়। একটা নির্ধারিত, নিয়ন্ত্রিত experiment-এর সময় ইচ্ছাকৃতভাবে এই ফাঁকগুলো খুঁজে পাওয়াই ঠিক এই exercise-এর মূল্য—একটা অনিয়ন্ত্রিত প্রকৃত incident-এর সময় এগুলো আবিষ্কার করার চেয়ে সেখানে আবিষ্কার করা অনেক ভালো।

**৮. "Game day" কী, এবং এটা কী টেস্ট করে যা শুধু একটা automated chaos experiment করে না?**
একটা game day হলো একটা নির্দিষ্ট failure scenario সিমুলেট করা একটা নির্ধারিত exercise যেখানে একটা team প্রকৃত incident response প্র্যাকটিস করে—শুধু automated system recover করে কিনা তা পর্যবেক্ষণ করা নয়, বরং on-call engineer সঠিক dashboard খুঁজে পেতে পারে কিনা, runbook সঠিকভাবে অনুসরণ করতে পারে কিনা, এবং সিমুলেটেড চাপের মধ্যে যথাযথভাবে escalate করতে পারে কিনা। এটা সিস্টেমের চারপাশের মানুষ এবং process টেস্ট করে, যা একটা শুধু automated chaos experiment কভার করে না।

**৯. Scenario: একটা team-এর ডিজাইন ডকুমেন্ট বলে database failover ১০ সেকেন্ডের মধ্যে সম্পন্ন হওয়া উচিত, কিন্তু এটা কখনো বাস্তবসম্মতভাবে টেস্ট করা হয়নি। আপনি কী সুপারিশ করবেন, এবং কেন?**
একটা নির্দিষ্ট hypothesis "ন্যূনতম latency প্রভাব সহ failover ১০ সেকেন্ডের মধ্যে সম্পন্ন হয়" নিয়ে একটা chaos experiment চালান, একটা bounded blast radius-এর মধ্যে, metric দেখার সময় একটা নির্ধারিত window-এ ইচ্ছাকৃতভাবে primary replica মেরে ফেলুন—এটা সরাসরি টেস্ট করে documented ডিজাইনের ধারণা বাস্তবে টিকে থাকে কিনা, একটা প্রকৃত failure এর উল্টোটা প্রমাণ না করা পর্যন্ত এটা ধরে নিতে থাকার বদলে।

**১০. সত্য নাকি মিথ্যা: একটা পরিচিত high-traffic event-এর আগে (যেমন একটা বড় sales day) একটা load test চালানো মূলত সিস্টেম কাজ করে তা নিশ্চিত করার জন্য, এবং কোনো সমস্যা না পাওয়াই সবচেয়ে ভালো ফলাফল।**
মূলত মিথ্যা—একটা পরিচিত event-এর আগে একটা load test-এর আসল মূল্য হলো capacity সমস্যা খুঁজে বের করা (যেমন একটা downstream service প্রত্যাশিত peak-এর নিচেই timeout শুরু করা) যখন সেটা ঠিক করার জন্য এখনও সময় আছে। একটা load test যা কিছু খুঁজে পায় না তার মানে হতে পারে সিস্টেম সত্যিই প্রস্তুত, কিন্তু একটা test যা কখনো কিছু খুঁজে পায় না তা নিয়ে সন্দিহান থাকা মূল্যবান, কারণ এটা হয়তো যথেষ্ট জোরে চাপ দিচ্ছে না বা সঠিক scenario কভার করছে না।
