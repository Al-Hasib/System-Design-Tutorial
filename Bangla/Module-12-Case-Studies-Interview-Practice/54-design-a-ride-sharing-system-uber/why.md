# কেন এই বিষয়টি গুরুত্বপূর্ণ: একটি Ride-Sharing System ডিজাইন করা (Uber-এর মতো)

> **এক বাক্যে:** এটিই capstone, কারণ কোর্সের আর কিছুই একসাথে এত কিছু একত্রিত করে না — বিশাল আয়তনে ক্রমাগত location write, geospatial query, প্রকৃত exclusivity constraint সহ একটি matching সমস্যা, দুটি live client জুড়ে real-time state, এবং টাকা।

## এই Case Study কেন বিদ্যমান

অন্য প্রতিটি case study-র একটি প্রধান বৈশিষ্ট্য আছে: একটি URL shortener read-heavy key-value, একটি feed fan-out, streaming হলো bandwidth। Ride-sharing-এ একসাথে একাধিক কঠিন সমস্যা আছে, এবং সেগুলো পরস্পর interact করে।

Driver-রা প্রতি কয়েক সেকেন্ডে location update পাঠায়, ক্রমাগত, কেউ দেখছে কি না তা নির্বিশেষে — একটি write volume যা প্রকৃত ride-এর সংখ্যাকে বহুগুণে ছাড়িয়ে যায়। Rider-দের দরকার "এই মুহূর্তে আমার কাছে কে আছে", যা এমন একটি query type যা সাধারণ index দক্ষতার সাথে উত্তর দিতে পারে না। Matching-কে একজন driver-কে একজন rider-এর সাথে *ঠিক একবার* assign করতে হবে, যা প্রকৃত টাকা জড়িত একটি distributed exclusivity সমস্যা। এবং একবার match হয়ে গেলে, উভয় পক্ষেরই trip চলাকালীন live update দরকার।

এটি এমন একটি case study যা "ভিন্ন data-র ভিন্ন প্রয়োজনীয়তা আছে" বলার সবচেয়ে বেশি পুরস্কার দেয়, কারণ একটি একক product-এর মধ্যেই এমন data আছে যা অবশ্যই strongly consistent হতে হবে (payment, ride assignment) এবং এমন data আছে যা স্পষ্টভাবে তা হওয়া উচিত নয় (একজন driver-এর তিন সেকেন্ড আগের location)।

## যে ডিজাইন সমস্যাগুলো এটি আপনাকে সমাধান করতে বাধ্য করে

### ১. এমন আয়তনে location write যা সবকিছুর উপর প্রাধান্য বিস্তার করে
**সমস্যা:** দশ লক্ষ active driver প্রতি চার সেকেন্ডে রিপোর্ট করলে তা প্রতি সেকেন্ডে আড়াই লক্ষ write, ক্রমাগত, যার বেশিরভাগ কখনো পড়া হবে না।

**কেন এটি কঠিন:** durability guarantee সহ একটি general-purpose database-এ এটি লেখা যুক্তিসঙ্গত খরচে অসম্ভব এবং অপ্রয়োজনীয় উভয়ই।

**যা আপনি শিখবেন:** প্রয়োজনীয়তাকে প্রশ্ন করা। Location ক্ষণস্থায়ী — ত্রিশ সেকেন্ড আগের একটি position মূল্যহীন — তাই সেগুলো একটি durable store-এ নয়, বরং একটি TTL সহ memory-তে (Redis) থাকা উচিত। ঐতিহাসিক track, billing বা dispute resolution-এর জন্য দরকার হলে, একটি queue-এর মাধ্যমে asynchronously একটি time-series বা columnar store-এ যায়। Hot path (বর্তমান position, memory-তে) থেকে cold path (trip history, batched to durable storage)-কে আলাদা করা প্রথম ও সবচেয়ে গুরুত্বপূর্ণ কাঠামোগত সিদ্ধান্ত।

### ২. দক্ষতার সাথে কাছাকাছি driver খুঁজে বের করা
**সমস্যা:** "এই পয়েন্টের 3 km-এর মধ্যে কোন driver-রা আছে?" এর উত্তর latitude ও longitude-এর উপর একটি B-tree দিয়ে দেওয়া যায় না — সেই index একটি dimension সংকীর্ণ করতে পারে, দুটি নয়।

**যা আপনি শিখবেন:** দুটি dimension-কে একটিতে হ্রাস করে **Geospatial indexing**। Geohash একটি location-কে একটি string prefix হিসেবে এনকোড করে যেখানে কাছাকাছি পয়েন্টগুলো prefix ভাগ করে, একটি proximity query-কে একটি prefix scan-এ পরিণত করে। Uber-এর H3 hexagonal cell ব্যবহার করে (hexagon-এর সব প্রতিবেশীর সাথে সমান দূরত্ব থাকে, square-এর মতো নয়)। উভয়ই "কাছাকাছি"-কে একটি scan-এর বদলে cell-এর একটি ছোট সেটে lookup-এ পরিণত করে। আপনি এটিকে বাস্তব করে তোলা edge case-গুলোও শিখবেন: একটি cell সীমানার ঠিক ওপারে থাকা একজন driver বাস্তবে কাছাকাছি কিন্তু index-এ নয়, তাই আপনি চারপাশের ring of cell-ও কোয়েরি করেন; এবং cell size টিউন করতে হয়, কারণ ঘন downtown cell এবং কম-ঘনত্বের গ্রামীণ cell-এর occupancy খুবই ভিন্ন।

### ৩. ঠিক একজন rider-কে একজন driver assign করা
**সমস্যা:** তিনজন rider একই সাথে request করেন এবং একই কাছাকাছি driver তিনজনের জন্যই সেরা match। প্রতিটি request একটি ভিন্ন server দ্বারা পরিচালিত হয়।

**কেন এটি কঠিন:** এটি একটি distributed mutual-exclusion সমস্যা, এবং এটি ভুল করলে একজন driver দুটি accepted ride পান।

**যা আপনি শিখবেন:** সবচেয়ে শক্তিশালী সমাধান সাধারণত একটি distributed lock নয়, বরং একটি **conditional atomic update** — driver-কে এমন একটি update দিয়ে claim করুন যা কেবল সফল হয় যদি তার status এখনো `available` থাকে, এবং database-এর নিজস্ব guarantee-কে কাজটি করতে দিন। যেখানে সত্যিকারের একটি lock দরকার, সেখানে topic 40-এর fencing-token পদ্ধতি প্রয়োজন। আপনি এর চারপাশের state machine-ও শিখবেন: একটি সংক্ষিপ্ত timeout সহ একটি offer, প্রত্যাখ্যাত বা উত্তরহীন হলে পরবর্তী candidate-এ একটি fallback, এবং idempotent handling যাতে একটি retry করা accept দুটি ride তৈরি না করে।

### ৪. একই সময়ে দুই পক্ষের জন্য real-time state
**সমস্যা:** একটি trip চলাকালীন, rider driver-কে এগিয়ে আসতে দেখেন এবং driver navigation ও status পরিবর্তন পান। উভয়েরই পুরো trip জুড়ে সেকেন্ডের মধ্যে update দরকার।

**যা আপনি শিখবেন:** উভয় পক্ষের জন্য persistent connection (WebSocket), chat-এর মতোই একই cross-server routing সমস্যা সহ — একটি location update একটি server-এ এসে পৌঁছালে তা অন্য একটি server-এর ধরে রাখা rider-এর socket-এ পৌঁছাতে হবে, যার মানে একটি Pub/Sub layer এবং একটি session registry। আপনি ইচ্ছাকৃতভাবে throttle করতেও শিখবেন: প্রতিটি raw GPS update rider-এর কাছে push করা অপচয়মূলক, তাই আপনি client-এ downsample এবং interpolate করেন। এটি একটি ভালো উদাহরণ যেখানে একটি product-quality সিদ্ধান্ত infrastructure load কমায়।

### ৫. একটি product-এর মধ্যে ভিন্ন consistency প্রয়োজনীয়তা
**সমস্যা:** একটি একক consistency standard প্রয়োগ করলে system হয় ভুল অথবা অসাশ্রয়ী হয়ে যায়।

**যা আপনি শিখবেন:** স্পষ্টভাবে শ্রেণীবদ্ধ করা। Payment এবং fare calculation অবশ্যই strongly consistent এবং transactional হতে হবে। Ride assignment অবশ্যই exclusive হতে হবে। Driver location কয়েক সেকেন্ড পুরনো হলেও কোনো পরিণতি নেই। Surge pricing aggregated demand থেকে হিসাব করা হয় এবং স্বভাবতই আনুমানিক। এটি জোরে বলা — "এই তিনটি জিনিসের ভিন্ন guarantee দরকার, এবং এই হলো কারণ" — পুরো কোর্স যে judgment তৈরি করার দিকে কাজ করছে তার সবচেয়ে স্পষ্ট প্রদর্শন।

## ভুল করলে যা খরচ হয়

- **প্রতিটি location update স্থায়ীভাবে (durably) সংরক্ষণ করলে** এমন একটি ডিজাইন তৈরি হয় যা প্রয়োজনের চেয়ে বহুগুণ বেশি ব্যয়বহুল।
- **Latitude ও longitude-কে প্রচলিতভাবে index করে** proximity-এর জন্য scan করা স্কেল করে না, এবং geospatial indexing পুরোপুরি বাদ দেওয়া এই interview-এর সবচেয়ে সাধারণ ফাঁক।
- **Double-assignment উপেক্ষা করলে** product-এর মূল correctness সমস্যাটি অমীমাংসিত থেকে যায়।
- **সর্বত্র strong consistency প্রয়োগ করলে** location pipeline তৈরিযোগ্য থাকে না।
- **Geospatial index-এ cell-boundary প্রভাব ভুলে গেলে** এমন একটি matcher তৈরি হয় যা সবচেয়ে কাছের driver-কে মিস করে।

## কেন Interviewer-রা এটি বেছে নেন

এটি সাধারণভাবে ব্যবহৃত সবচেয়ে ব্যাপক সমস্যা, যা এটিকে একটি স্বাভাবিক capstone এবং interviewer যেদিকে চান সেদিকে গভীরতা যাচাইয়ের একটি নির্ভরযোগ্য উপায় করে তোলে। একজন candidate-কে write pipeline, geospatial index, matching algorithm, real-time layer, বা payment path-এর দিকে ঠেলে দেওয়া যায় — এবং প্রতিটিই একটি বৈধ deep dive। এটি এই কোর্স জুড়ে গড়ে তোলা দুটি অভ্যাসকেও পুরস্কৃত করে: ডিজাইন করার আগে estimate করুন, এবং প্রতি system-এর বদলে প্রতি use case-এর জন্য guarantee বেছে নিন।

## এটি কীভাবে সংযুক্ত (How It Connects)

এই case study একত্রিত করে সাধারণ **indexing** নীতির (topic 12) উপর **geospatial indexing**, in-memory location state-এর জন্য **Redis** (topic 19), asynchronous location ও trip pipeline-এর জন্য **message queue** (topic 20), surge pricing ও demand aggregation-এর জন্য **stream processing** (topic 23), live trip update-এর জন্য **Pub/Sub** routing সহ **WebSocket** (topic 10, 21), matching-এর জন্য **distributed locking** ও **atomic conditional update** (topic 40, 37), retried request ও payment-এর জন্য **idempotency** (topic 29), ভূগোল অনুযায়ী **sharding** (topic 14), use case অনুযায়ী প্রয়োগ করা **consistency model** (topic 29), এবং ride-এবং-payment lifecycle-এর জন্য saga-ভিত্তিক flow সহ **microservices** (topic 30, 28)।

**এখান থেকে কোথায় যাবেন:** আপনি এখন "system design কী" থেকে শুরু করে এমন একটি system পর্যন্ত পুরো পথ হেঁটেছেন যা প্রায় প্রতিটি concept প্রয়োগ করে। সবচেয়ে কার্যকর পরবর্তী ধাপ হলো এই case study-গুলোর দুই বা তিনটি একটি শূন্য পাতা থেকে, জোরে বলে, একটি timer সহ পুনরায় করা — একটি argument চেনার এবং চাপের মধ্যে সেটি তৈরি করার মধ্যকার ব্যবধানই আসলে interview performance যেখানে বাস করে।
