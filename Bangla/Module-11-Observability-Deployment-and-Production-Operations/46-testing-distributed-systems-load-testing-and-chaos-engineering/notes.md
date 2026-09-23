# Study Notes: Testing Distributed Systems

## সংজ্ঞা

- **Load testing:** সিস্টেম টিকে থাকে কিনা যাচাই করতে প্রত্যাশিত peak level-এ synthetic traffic তৈরি করা।
- **Stress testing:** breaking point খুঁজে বের করতে এবং failure mode পর্যবেক্ষণ করতে (graceful degradation বনাম cascading failure) প্রত্যাশিত peak-এর অনেক বেশি traffic পাঠানো।
- **Soak testing (endurance testing):** ধীর leak এবং ধীরে ধীরে degradation ধরতে দীর্ঘ সময় ধরে একটা sustained, moderate load চালানো।
- **Chaos engineering:** একটা সিস্টেমে ইচ্ছাকৃতভাবে বাস্তব failure inject করা যাতে confidence তৈরি হয় যে এর resilience mechanism আসলেই কাজ করে।
- **Blast radius:** একটা experiment-কে প্রভাবিত করতে দেওয়া bounded scope (traffic-এর শতাংশ, নির্দিষ্ট environment)।
- **Game day:** একটা নির্দিষ্ট failure scenario সিমুলেট করা একটা নির্ধারিত exercise, automated recovery এবং human incident response দুটোই টেস্ট করতে।

## Load Testing-এর ধরন

| ধরন | Traffic level | যে প্রশ্নের উত্তর দেয় |
|---|---|---|
| Load test | প্রত্যাশিত peak | আমরা যে traffic প্রকৃতপক্ষে প্রত্যাশা করি তাতে এটা কি টিকে থাকে? |
| Stress test | প্রত্যাশিত peak-এর অনেক বেশি | Breaking point কোথায়, আর এটা কি গ্রেসফুলি নাকি catastrophically ব্যর্থ হয়? |
| Soak test | Moderate, ঘণ্টা/দিনের পর দিন sustained | এটা কি সময়ের সাথে সাথে ধীরে degrade হয় (leak, resource exhaustion)? |

সাধারণ টুল: k6, Locust, Gatling, Apache JMeter।

## Chaos Engineering Practice

1. **একটা hypothesis তৈরি করুন:** যেমন, "যদি primary DB replica ব্যর্থ হয়, ন্যূনতম latency প্রভাব সহ failover ১০ সেকেন্ডের মধ্যে সম্পন্ন হয়।"
2. **Blast radius নির্ধারণ করুন:** প্রথমে traffic-এর একটা ছোট শতাংশ, অথবা একটা non-critical/staging environment; observability metric-এর সাথে যুক্ত একটা automatic abort trigger সেট করুন।
3. **Experiment চালান:** team দেখার সময় একটা নির্ধারিত window-এ failure inject করুন (একটা instance মারুন, network latency যোগ করুন, একটা timeout সিমুলেট করুন)।
4. **পর্যবেক্ষণ করুন:** প্রকৃত আচরণ কি hypothesis-এর সাথে মেলে? (প্রায়ই মেলে না, কোনো নির্দিষ্ট উপায়ে—এটাই এই exercise-এর মূল্য।)
5. **ঠিক করুন এবং সম্প্রসারিত করুন:** যা পাওয়া গেছে তা সমাধান করুন, তারপর ভবিষ্যৎ experiment-এর blast radius/জটিলতা ধীরে ধীরে বাড়ান।

## Load Testing বনাম Chaos Engineering

| | Load Testing | Chaos Engineering |
|---|---|---|
| যা টেস্ট করে | বেশি/sustained traffic-এর অধীনে আচরণ | বাস্তব infrastructure failure-এর অধীনে আচরণ |
| যে প্রশ্নের উত্তর দেয় | Capacity/scaling কি টিকে থাকে? | Resilience mechanism (retries, failover, circuit breakers) কি আসলেই কাজ করে? |
| সাধারণ টুল | k6, Locust, Gatling | Chaos Monkey, Gremlin, Litmus |

## Game Days

- শুধু automated system নয়, মানুষ এবং runbook টেস্ট করে।
- একটা নির্দিষ্ট scenario (region outage, dependency failure, data corruption) সিমুলেট করে একটা নির্ধারিত, পরিকল্পিত exercise-এ।
- এমন ফাঁক প্রকাশ করে যেমন: on-call engineer কি সঠিক dashboard খুঁজে পেতে পারে, escalation process কি কাজ করে, runbook কি আসলেই সঠিক/হালনাগাদ।

## গুরুত্বপূর্ণ সংখ্যা / তথ্য

- Netflix-এর Chaos Monkey (বৃহত্তর "Simian Army"-এর অংশ) প্রথম দিকের ব্যাপকভাবে প্রচারিত production chaos engineering টুলগুলোর একটা ছিল, যা প্রায় ২০১১ সালে চালু হয়েছিল।
- Principles of Chaos Engineering (chaosprinciples.org) hypothesis-driven, blast-radius-controlled approach-কে একটা industry discipline হিসেবে formalize করে।

## সারাংশ

- Unit/integration test আলাদা করে logic যাচাই করে; load testing এবং chaos engineering যথাক্রমে বাস্তব traffic এবং বাস্তব failure-এর অধীনে আচরণ যাচাই করে।
- Load/stress/soak testing প্রতিটি traffic আচরণ সম্পর্কে একটা আলাদা প্রশ্নের উত্তর দেয়; chaos engineering উত্তর দেয় resilience mechanism বাস্তব, ইচ্ছাকৃতভাবে-inject করা failure-এর অধীনে আসলেই কাজ করে কিনা।
- Game day এটাকে সম্প্রসারিত করে human response এবং runbook টেস্ট করার জন্য—একটা perfect automated design-ও ব্যর্থ হতে পারে যদি এর চারপাশের incident response কাজ না করে।
