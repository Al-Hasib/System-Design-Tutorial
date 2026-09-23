# Why This Topic Matters: Testing Distributed Systems — Load Testing & Chaos Engineering

> **এক বাক্যে:** Unit test প্রমাণ করে আপনার function সঠিক; এগুলো কিছুই বলে না যখন traffic ৫০ গুণ বেড়ে যায় বা একটা dependency উত্তর দেওয়া বন্ধ করে দেয়—আর এই দুটোই আসলে সিস্টেমকে ধসিয়ে দেয়।

## এই আইডিয়ার আগের জগৎ

**নির্ধারিত সময়েই ব্যর্থ হওয়া লঞ্চ।** সকাল ৯টায় একটা campaign লাইভ হয়। Traffic স্বাভাবিকের ৪০ গুণ, ঠিক যেমনটা marketing বলেছিল। চার মিনিটের মধ্যে সাইট ডাউন হয়ে যায়। Connection pool স্বাভাবিক load-এর জন্য সাইজ করা ছিল, একটা unindexed query concurrency-তে fatal হয়ে ওঠে, আর autoscaling যে capacity যোগ করতে ৬ মিনিট নিল ততক্ষণে অনেক দেরি হয়ে গেছে। প্রতিটা component-এর ১০০% test coverage ছিল। কিন্তু এর কোনোটাই কখনো load-এর অধীনে চালানো হয়নি।

**যে failover কখনো চেষ্টা করে দেখা হয়নি।** একটা database primary ব্যর্থ হয়। Team-এর কাছে একটা replica এবং আঠারো মাস আগে লেখা একটা documented failover procedure আছে। চাপের মধ্যে তারা আবিষ্কার করে replica-র configuration drift হয়ে গেছে, promotion script একটা decommissioned host-কে reference করছে, এবং application connection string গুলো পুরনো primary-তে hardcode করা। যেটা দুই মিনিটের failover হওয়ার কথা ছিল সেটা তিন ঘণ্টার outage হয়ে যায়। Redundancy ছিল; শুধু সেটা কখনো exercise করা হয়নি।

দুটো failure-এরই একটা common কারণ আছে: গুরুত্বপূর্ণ পরিস্থিতিতে সিস্টেমের আচরণ কখনো পর্যবেক্ষণ করা হয়নি, শুধু ধরে নেওয়া হয়েছিল।

## এটা যেসব সমস্যার সমাধান করে

### ১. Capacity যা জানা নয়, শুধু অনুমান করা
**আপনি যা দেখেন:** "আমরা প্রায় ১০,০০০ requests per second সামলাতে পারি"—একটা সংখ্যা যা কেউ পরিমাপ করেনি, optimism থেকে বের করা।

**কেন এটা ঘটে:** Load characteristic emergent। Connection pool limit, thread exhaustion, lock contention, GC pressure, এবং একটা slow query যা শুধু concurrency-তে ক্ষতি করে—এগুলো একবারে একটা request পাঠালে অদৃশ্য থাকে।

**Load testing কীভাবে এটার সমাধান করে:** এটা একটা বাস্তব সংখ্যা তৈরি করে এবং, আরও কার্যকরীভাবে, প্রথমে কোন component ভাঙবে তা খুঁজে বের করে। ভিন্ন ভিন্ন test shape ভিন্ন ভিন্ন প্রশ্নের উত্তর দেয়: প্রত্যাশিত peak-এ **load testing** যাচাই করে আপনি আপনার SLO পূরণ করছেন কিনা; breaking point-এর ওপারে **stress testing** বলে দেয় limit কোথায় এবং এটা *কীভাবে* ব্যর্থ হয় (গ্রেসফুলি degrade করে, নাকি ভেঙে পড়ে); **spike testing** যাচাই করে autoscaling যথেষ্ট দ্রুত react করে কিনা; অনেক ঘণ্টা ধরে **soak testing** memory leak এবং connection leak খুঁজে বের করে যেগুলো শুধু সময়ের সাথে সাথে দেখা যায়।

### ২. Non-linear আচরণ যা আপনি extrapolate করতে পারবেন না
**আপনি যা দেখেন:** ১,০০০ থেকে ৪,০০০ requests per second পর্যন্ত latency সমতল থাকে, তারপর ৪,৫০০-এ গিয়ে হঠাৎ উলম্বভাবে বেড়ে যায়।

**কেন এটা ঘটে:** সিস্টেমে ঢাল নয়, খাড়া পাহাড় থাকে। একটা queue যা ঠিকঠাক চলছে সেটা ততক্ষণ ঠিক থাকে যতক্ষণ না এটা আর ঠিক থাকে না, আর তারপর এটা অসীমভাবে জমতে থাকে। Traffic দ্বিগুণ করলে latency দ্বিগুণ হয় না; এটা পঞ্চাশ গুণও হয়ে যেতে পারে।

**Testing কীভাবে এটার সমাধান করে:** শুধু measurement-ই সেই খাড়া পাহাড় খুঁজে পায়। এটা কোথায় আছে তা জানলে আপনি এর *আগেই* autoscaling threshold এবং alert সেট করতে পারবেন, একটা incident-এর সময় এটা আবিষ্কার করার বদলে।

### ৩. Failure-handling কোড যা কখনো চলেনি
**আপনি যা দেখেন:** Circuit breaker, retries, fallback, এবং failover script যেগুলো আছে, untested, এবং প্রথমবার exercise হওয়ার সময়ই ব্যর্থ হয়—একটা প্রকৃত incident-এর সময়।

**কেন এটা ঘটে:** এই কোড শুধু তখনই চলে যখন কিছু একটা ভেঙে যায়, যা ডিজাইন অনুযায়ী testing-এ ঘটে না। এটা আপনার সবচেয়ে কম exercise করা কোড এবং যে কোড আপনার সবচেয়ে বেশি কাজ করার দরকার।

**Chaos engineering কীভাবে এটার সমাধান করে:** ইচ্ছাকৃতভাবে failure inject করুন—একটা instance মেরে ফেলুন, একটা dependency-তে ৫০০ ms latency যোগ করুন, একটা শতাংশ packet drop করুন, network partition করুন, একটা disk exhaust করুন—এবং যাচাই করুন সিস্টেম ডিজাইন অনুযায়ী আচরণ করে কিনা। কাজের সময়ে এটা করুন, team দেখছে এবং থামানোর একটা উপায় আছে এমন অবস্থায়, যাতে আপনি নিজের শর্তে শিখতে পারেন, রাত ৩টায় নয়। উদ্দেশ্য জিনিস ভাঙা নয়; এটা হলো *failure সম্পর্কে আপনার ধারণাগুলো সত্যি কিনা তা নিশ্চিত করা*, আর প্রায়ই সেগুলো সত্যি হয় না।

### ৪. অজানা dependency এবং লুকানো coupling
**আপনি যা দেখেন:** একটা "non-critical" service বন্ধ করলে checkout ডাউন হয়ে যায়, আর কেউ এটা predict করেনি।

**কেন এটা ঘটে:** অনেক service-এর একটা সিস্টেমে, প্রকৃত dependency graph যেকারো মাথায় থাকা graph থেকে আলাদা হয়ে যায়। সময়ের সাথে যোগ হওয়া synchronous call নিঃশব্দে ঐচ্ছিক জিনিসকে বাধ্যতামূলক করে তোলে।

**Chaos experiment কীভাবে এটার সমাধান করে:** এগুলো প্রকৃত graph প্রকাশ করে। "আমরা recommendations বন্ধ করলাম আর checkout ভেঙে গেল" একটা আবিষ্কার যা মঙ্গলবার বিকেলে সস্তা আর Black Friday-তে বিপর্যয়কর।

## যে মূল্য আপনাকে দিতে হয়

- **বাস্তবসম্মত load test তৈরি করা কঠিন।** Synthetic traffic যা প্রকৃত access pattern-এর সাথে মেলে না তা বিভ্রান্তিকর ফলাফল দেয়—একটা অবাস্তবভাবে ছোট key set আপনাকে ১০০% cache hit rate দেবে এবং একটা capacity সংখ্যা দেবে যা প্রকৃতের চেয়ে কয়েকগুণ বেশি। ভালো load test-এর জন্য production-এর মতো data distribution দরকার, যেটা তৈরি করতে পরিশ্রম লাগে।
- **Test environment মিথ্যা বলে।** এক-দশমাংশ data এবং ভিন্ন hardware থাকা একটা staging environment production-এর সমস্যাগুলো সামনে আনবে না। Production-এ testing করা বেশি নির্ভুল এবং অনেক বেশি সাবধানতা দাবি করে।
- **Production-এ chaos প্রকৃতপক্ষে ঝুঁকিপূর্ণ।** এর জন্য এমন কিছু prerequisite দরকার যেগুলো বেশিরভাগ team underestimate করে: শক্তিশালী observability, একটা ছোট নিয়ন্ত্রিত blast radius, একটা automatic abort, এবং organizational buy-in। Staging-এ শুরু করাই সঠিক প্রথম পদক্ষেপ, আর সরাসরি production experiment-এ ঝাঁপিয়ে পড়া হলো একটা কোম্পানিতে chaos engineering নিষিদ্ধ হয়ে যাওয়ার উপায়।
- **এটা infrastructure এবং সময় খরচ করে।** বড় স্কেলে load generation বিনামূল্যে নয়, এবং test harness তৈরি ও maintain করা একটা চলমান কাজ যা feature-এর সাথে প্রতিযোগিতা করে।
- **ফলাফল পুরনো হয়ে যায়।** একটা capacity সংখ্যা শুধু সেই code এবং configuration-এর জন্য বৈধ যেটা এটা তৈরি করেছে। নিয়মিত পুনরায় না চালালে, এটা কিংবদন্তি হয়ে যায়—একটা untested failover runbook-এর মতোই একই সমস্যা।

## কখন এটা দরকার — এবং কখন নয়

| এতে বিনিয়োগ করুন যখন | হালকা approach ঠিক আছে যখন |
|---|---|
| একটা পরিচিত traffic event আসছে (launch, sale, campaign) | Traffic ছোট, স্থিতিশীল, এবং কোনো limit থেকে অনেক দূরে |
| Availability commitment চুক্তিভিত্তিক | ধৈর্যশীল ব্যবহারকারী থাকা একটা internal tool |
| Architecture-এ অনেক service এবং failure mode আছে | একটা dependency থাকা একটা একক service |
| আপনার এমন redundancy আছে যা কখনো আসলে exercise করা হয়নি | — |

**আকার নির্বিশেষে একটা জিনিস করার মতো: আপনার failover এবং restore procedure আসলেই টেস্ট করুন।** একটা untested backup কোনো backup নয়, আর একটা untested failover কোনো redundancy নয়।

## কেন এটা Interview-এ আসে

Interviewer জিজ্ঞেস করেন "আপনি কীভাবে জানবেন এই ডিজাইন load সামলাতে পারবে?" এবং "আপনি কীভাবে যাচাই করবেন এটা একটা failure টিকিয়ে রাখতে পারবে?" তারা দেখতে চান আপনার capacity সংখ্যাগুলো ভিত্তিযুক্ত নাকি আবিষ্কৃত। শক্তিশালী candidate তাদের নিজস্ব estimation-এর সাথে যুক্ত করেন—"আমি ৮,০০০ writes per second অনুমান করেছিলাম, তাই আমি load test চালাতাম নিশ্চিত করতে যে একটা single shard তার ভাগ সামলাতে পারে এবং কোথায় এটা ভাঙে তা খুঁজে বের করতে"—এবং resilience-কে অনুমান করার বদলে যাচাই করার মতো কিছু হিসেবে দেখেন। আপনি এইমাত্র বর্ণনা করা circuit breaker সত্যিই কাজ করে কিনা তা নিশ্চিত করতে একটা chaos experiment চালাবেন উল্লেখ করা—এটা এমনভাবে লুপটা বন্ধ করে যা বিরল এবং স্মরণীয়।

## এটা কীভাবে যুক্ত

Testing কোর্সের প্রায় বাকি সবকিছু validate করে: requirements থেকে **capacity estimates** (topic 2), **horizontal scaling** এবং autoscaling আচরণ (topic 4), **fault tolerance** দাবি (topic 5), **circuit breakers এবং retries** (topic 26), **replication failover** (topic 13), এবং **multi-region disaster recovery** (topic 47)। **observability** (topic 43) ছাড়া এটা অসম্ভব, যা একটা experiment-কে একটা measurement-এ পরিণত করে, এবং এটা স্বাভাবিকভাবেই production থেকে নিরাপদে শেখার আরেকটা উপায় হিসেবে **canary deployments** (topic 45)-এর সাথে জোড়া লাগে।

**পরবর্তী:** [Multi-Region Architecture & Disaster Recovery](../47-multi-region-architecture-and-disaster-recovery/why.md) — একটা পুরো datacenter-এর ব্যর্থতা টিকিয়ে রাখা।
