# এই বিষয়টি কেন গুরুত্বপূর্ণ: একটি নিউজ ফিড সিস্টেম ডিজাইন করা (Twitter/Facebook-এর মতো)

> **এক বাক্যে:** এটি এমন একটি সমস্যা যেখানে একটি মাত্র সিদ্ধান্ত — কেউ লেখার সময় feed compute করবেন, নাকি কেউ পড়ার সময় — পুরো architecture নির্ধারণ করে দেয়, এবং যেখানে সঠিক উত্তর হলো "উভয়ই, ব্যবহারকারীর উপর নির্ভর করে", আর ঠিক এই কারণেই এটি system design-এর সেরা trade-off প্রশ্ন।

## এই কেস স্টাডিটি কেন আছে

একটি news feed দেখতে একটি সাধারণ query-এর মতো মনে হয়: যাদের আমি follow করি তাদের সাম্প্রতিক post নিন, সময় অনুযায়ী সাজান। ছোট একটি dataset-এ এটি ঠিক সেরকম query-ই।

কিন্তু স্কেলে এটি অসম্ভব হয়ে ওঠে। ২,০০০ account follow করা একজন ব্যবহারকারী প্রতিবার feed refresh করার সময় ২,০০০টি partition জুড়ে একটি query ট্রিগার করে, যা sort ও merge করতে হয়, এবং তাও কোটি কোটি ব্যবহারকারীর জন্য। স্পষ্ট সমাধান — একটি post তৈরি হওয়ার সময়ই সবার feed precompute করা — সুন্দরভাবে কাজ করে, যতক্ষণ না ১০ কোটি follower-বিশিষ্ট কেউ post করে, আর তখন একটি write ১০ কোটি write-এ পরিণত হয়।

উভয় প্রান্তই ব্যর্থ হয়। ডিজাইন হলো এই দুইয়ের মধ্যে একটি negotiation, এবং এটিই এই অনুশীলনের মূল বিষয়।

## যে ডিজাইন সমস্যাগুলো এটি আপনাকে সমাধান করতে বাধ্য করে

### ১. Fan-out on write বনাম fan-out on read
**সমস্যা:** Feed assembly ব্যয়বহুল, এবং আপনাকে কোথাও না কোথাও এর মূল্য দিতেই হবে।

**কেন এটি কঠিন:** এই দুটি option-এর cost profile এবং failure mode একেবারে বিপরীত।

**যা শিখবেন:** **Fan-out on write** (push): একজন ব্যবহারকারী post করলে, post ID প্রতিটি follower-এর precomputed feed list-এ append করা হয়। Reads হয়ে যায় একটি মাত্র দ্রুত lookup — একটি read-heavy প্রোডাক্টের জন্য নিখুঁত — কিন্তু writes follower সংখ্যা দ্বারা amplify হয়, এবং একজন celebrity-র post একটি write storm-এ পরিণত হয়। **Fan-out on read** (pull): post একবার সংরক্ষণ করুন এবং request-এর সময় feed assemble করুন। Writes সস্তা ও constant; reads ব্যয়বহুল ও ধীর, বিশেষত যারা অনেক account follow করে তাদের জন্য।

**যে সমন্বয় (synthesis) এই সমস্যাটিকে বিখ্যাত করে তোলে:** সাধারণ ব্যবহারকারীদের জন্য fan-out on write ব্যবহার করুন, এবং বিশাল সংখ্যক follower-বিশিষ্ট অল্প কয়েকটি account-এর জন্য fan-out on read ব্যবহার করুন। তখন একজন ব্যবহারকারীর feed হয়ে যায় তার precomputed list-এর সাথে সেই হাতেগোনা কয়েকজন celebrity-র জন্য একটি live query-এর merge, যাদের তিনি follow করেন। এই hybrid-এ পৌঁছানো — এবং কেন কোনো pure পদ্ধতি কাজ করে না তা ব্যাখ্যা করা — এই ইন্টারভিউয়ে সবচেয়ে বেশি মূল্যবান একক উত্তর।

### ২. Storage যা content-এর চেয়ে দ্রুত বৃদ্ধি পায়
**সমস্যা:** Precomputed feed প্রতিটি follower জুড়ে post reference-গুলো ডুপ্লিকেট করে, তাই storage বাড়ে *follow graph*-এর সাথে সাথে, post-এর সংখ্যার সাথে নয়।

**যা শিখবেন:** প্রতিটি মিটিগেশনের নিজস্ব একটি cost আছে। Post content-এর বদলে post ID সংরক্ষণ করুন এবং read সময়ে hydrate করুন (কম storage, একটি অতিরিক্ত lookup)। প্রতিটি feed কয়েকশ এন্ট্রিতে সীমাবদ্ধ রাখুন, কারণ প্রায় কেউই এর বেশি scroll করে না। নিষ্ক্রিয় (inactive) ব্যবহারকারীদের জন্য fan-out সম্পূর্ণ এড়িয়ে যান এবং তারা ফিরে এলে on demand তাদের feed compute করুন — এটি সত্যিকারের একটি বড় সাশ্রয়, কারণ নিষ্ক্রিয় account-এর long tail-ই ব্যবহারকারীদের অধিকাংশ।

### ৩. Feed generation যা post request-কে block করতে পারে না
**সমস্যা:** একজন ব্যবহারকারী post করে এবং তাৎক্ষণিক নিশ্চিতকরণ পেতে হয়, কিন্তু ১০ লাখ follower-এর কাছে fan out করতে সময় লাগে।

**যা শিখবেন:** Write path post-টি persist করে এবং একটি event publish করে; worker-রা তা consume করে এবং asynchronous-ভাবে fan-out করে। এতে feed **eventually consistent** হয়ে যায় — একজন follower কয়েক সেকেন্ডের জন্য post-টি নাও দেখতে পারে — এবং এটি আপনাকে শেখায় যাচাই করতে যে এটি গ্রহণযোগ্য কিনা (একটি social feed-এর জন্য, অবশ্যই হ্যাঁ; একটি stock price-এর জন্য, অবশ্যই না)। এটি consumer lag পর্যবেক্ষণের operational বাস্তবতাও তুলে ধরে, কারণ একটি backed-up fan-out queue-ই হলো যেভাবে feed নীরবে stale হয়ে যায়।

### ৪. Ranking, এবং কেন এটি data model পরিবর্তন করে
**সমস্যা:** একটি chronological feed হলো একটি sorted merge। একটি ranked feed ("top posts for you") engagement, recency, affinity, এবং একটি model দিয়ে candidate স্কোর করা প্রয়োজন।

**যা শিখবেন:** Ranking read path-কে candidate generation ও scoring-এ বিভক্ত করে, একটি feature store এবং model-serving dependency যোগ করে, এবং latency বাজেটকে অনেক আঁটোসাঁটো করে তোলে। এটি pagination-কেও কঠিন করে তোলে: একটি ক্রমাগত পরিবর্তনশীল ranked list-এ, offset-based paging ডুপ্লিকেট দেখায় এবং item এড়িয়ে যায়, এবং সে কারণেই **cursor-based pagination** সঠিক পছন্দ — একটি ছোট, নির্দিষ্ট বিস্তারিত যা ইন্টারভিউয়াররা লক্ষ্য করেন।

### ৫. Read volume যা বাকি সবকিছুকে ছাপিয়ে যায়
**সমস্যা:** যেকোনো consumer product-এ Feed reads সর্বোচ্চ-volume অপারেশনগুলোর একটি।

**যা শিখবেন:** এখানে caching কোনো অপ্টিমাইজেশন নয়, এটিই architecture। Precomputed feed Redis-এ থাকে; post content আলাদাভাবে cache করা হয় এবং hydrate করা হয়; media সম্পূর্ণভাবে একটি CDN থেকে সার্ভ করা হয়। এবং আপনি শিখবেন সেই estimation প্রশ্ন জিজ্ঞাসা করতে যা সবকিছু নির্ধারণ করে — daily active users গুণ refreshes per day ভাগ সেকেন্ড দিয়ে — কারণ এই সংখ্যাই ঠিক করে আপনার ডায়াগ্রামে কতগুলো cache node থাকবে।

## ভুল করলে যে মূল্য দিতে হয়

- **একটি মাত্র fan-out strategy-তে অটল থাকা** অন্যটির আলোচনা না করে — এই ইন্টারভিউয়ের সবচেয়ে সাধারণ দুর্বলতা, এবং এটিই ঠিক যা পরীক্ষা করা হচ্ছে।
- **Celebrity সমস্যা উপেক্ষা করা** এমন একটি ডিজাইন তৈরি করে যা ৯৯.৯৯% ব্যবহারকারীর জন্য কাজ করে কিন্তু সবচেয়ে বেশি ট্রাফিক তৈরি করা account-গুলোর জন্য ভেঙে পড়ে।
- **Post request-এ synchronous-ভাবে fan out করা** posting latency-কে follower সংখ্যার সাথে যুক্ত করে দেয়।
- **একটি ranked feed-এ Offset pagination** ডুপ্লিকেট ও অনুপস্থিত item তৈরি করে, যে bug ব্যবহারকারীরা তাৎক্ষণিক দেখতে পায়।
- **Storage amplification ভুলে যাওয়া** infrastructure cost-কে একটি order of magnitude কম করে দেখায়।

## ইন্টারভিউয়াররা কেন এটি বেছে নেন

এখানে একক কোনো সঠিক উত্তর নেই, যা এটিকে recall-এর বদলে judgment মূল্যায়নের একটি চমৎকার instrument করে তোলে। ইন্টারভিউয়ার যেকোনো দিকে চাপ দিতে পারেন — "যদি আমাদের ২০ কোটি follower-বিশিষ্ট একজন ব্যবহারকারী থাকে তাহলে কী হবে?", "যদি storage cost-ই constraint হয় তাহলে?", "যদি feed অবশ্যই ranked হতে হয় তাহলে?" — এবং candidate কীভাবে adapt করে তা দেখতে পারেন। এটি প্রায় প্রতিটি আগের বিষয়কেও একসাথে কাজে লাগায়: caching, sharding, queue, denormalization, eventual consistency, এবং CDN offload।

## এটি কীভাবে সংযুক্ত

এই কেস স্টাডি লোড-বেয়ারিং স্কেলে **caching** প্রয়োগ করে (topic 17), feed list-এর জন্য **Redis** data structure (topic 19), **message queue** এবং asynchronous **fan-out** (topics 20, 21), একটি ইচ্ছাকৃত read optimization হিসেবে **denormalization** (topic 16), **eventual consistency** (topic 29), post ও graph store-এর **sharding** (topic 14), media-এর জন্য **CDN** ডেলিভারি (topic 18), "এই ব্যবহারকারী কি ইতিমধ্যে এই post দেখেছে?"-এর জন্য **Bloom filter** (topic 42), এবং trending ও ranking signal-এর জন্য **batch versus stream processing** (topic 23)।

**পরবর্তী:** [একটি Distributed File Storage সিস্টেম ডিজাইন করা](../52-design-a-distributed-file-storage-google-drive/why.md) — যেখানে কঠিন অংশগুলো fan-out থেকে সরে গিয়ে chunking, deduplication, এবং sync conflict-এ চলে যায়।
