# এই বিষয়টি কেন গুরুত্বপূর্ণ: একটি ভিডিও স্ট্রিমিং প্ল্যাটফর্ম ডিজাইন করা (YouTube/Netflix-এর মতো)

> **এক বাক্যে:** এই স্কেলে অ্যাপ্লিকেশনটি একটি content delivery সমস্যার উপর একটি পাতলা স্তর মাত্র — architecture-টি মূলত বিশাল ফাইল ব্যবহারকারীদের কাছাকাছি নিয়ে যাওয়া এবং তাদের কাছে আসলে যে bandwidth আছে তার সাথে খাপ খাইয়ে নেওয়ার বিষয়ে।

## এই কেস স্টাডি কেন বিদ্যমান

বেশিরভাগ সিস্টেম requests per second দ্বারা bottleneck হয়। এটি bottleneck হয় **bits per second** দ্বারা, এবং এই পার্থক্যটি সবকিছু পুনর্বিন্যাস করে দেয়।

একটি HD ভিডিওর মাত্র এক ঘণ্টা কয়েক গিগাবাইট হয়ে যায়। লক্ষ লক্ষ concurrent viewer মানে সামগ্রিক egress-এ টেরাবিট প্রতি সেকেন্ড — এমন একটি পরিমাণ যা আপনি একটি origin থেকে সার্ভ করতে পারবেন না, standard egress pricing-এ বহন করতে পারবেন না, এবং একটি সমুদ্র জুড়ে গ্রহণযোগ্য latency-তে ডেলিভার করতে পারবেন না। CDN শেষে জুড়ে দেওয়া একটি অপ্টিমাইজেশন স্তর নয়; এটিই সিস্টেম। Netflix ঠিক এই কারণেই ISP-র data center-এ ফিজিক্যাল caching appliance পাঠায়।

এটি **write-once, read-astronomically-often** প্যাটার্নের এবং asynchronous processing pipeline-এরও সবচেয়ে স্পষ্ট কেস স্টাডি, কারণ upload-এর পর transcode হতে থাকা কয়েক মিনিট ধরে একটি ভিডিও দেখার অযোগ্য থাকে।

## এটি যে ডিজাইন সমস্যাগুলো সমাধান করতে বাধ্য করে

### ১. টেরাবিট-প্রতি-সেকেন্ড ডেলিভার করা
**সমস্যা:** Origin bandwidth এবং cost centralized serving-কে অসম্ভব করে তোলে, এবং transcontinental latency এটিকে ধীর করে দেয়।

**যা শিখবেন:** Multi-tier CDN architecture — জনপ্রিয় কনটেন্ট ব্যবহারকারীদের কাছাকাছি edge location-এ cache করা, কম জনপ্রিয় কনটেন্ট regional tier-এ, এবং সম্পূর্ণ catalog origin-এ। আপনি popularity distribution নিয়ে যুক্তি দিতে শিখবেন: কনটেন্টের একটি ছোট অংশ বেশিরভাগ view-এর জন্য দায়ী, তাই একটি সাধারণ edge cache খুব উচ্চ hit rate ধরতে পারে। এছাড়াও আপনি *pre-positioning* শিখবেন: একটি পরিচিত রিলিজের জন্য (একটি নতুন সিজন, একটি নির্ধারিত প্রিমিয়ার), প্রথম অনুরোধে cache warm করার পরিবর্তে চাহিদা আসার আগেই কনটেন্ট edge-এ পুশ করা।

### ২. এমন viewer-দের সার্ভ করা যাদের bandwidth সেকেন্ডে সেকেন্ডে পরিবর্তিত হয়
**সমস্যা:** একজন viewer ফাইবারে আছেন, আরেকজন একটি ট্রেনে একটি ওঠানামা-করা মোবাইল সংযোগে। একটি একক encoding দুজনের জন্যই ব্যর্থ হয়।

**যা শিখবেন:** **Adaptive bitrate streaming।** ভিডিওটিকে resolution এবং bitrate-এর একটি ladder-এ transcode করা হয়, প্রতিটি ছোট segment-এ (সাধারণত 2-10 সেকেন্ড) বিভক্ত। কী উপলব্ধ আছে তা তালিকাভুক্ত করে একটি manifest ফাইল। *ক্লায়েন্ট* নিজের throughput এবং buffer level পরিমাপ করে এবং পরবর্তী segment-এর জন্য কোন quality অনুরোধ করবে তা নির্বাচন করে, প্লেব্যাকের মাঝেই বাধাহীনভাবে পরিবর্তন করে। এই কারণেই অনির্ভরযোগ্য নেটওয়ার্কেও streaming কাজ করে, এবং এটি ব্যাখ্যা করে কেন server-পাশ তুলনামূলকভাবে সহজ: বুদ্ধিমত্তা ইচ্ছাকৃতভাবে player-এ রাখা হয়েছে।

এটি আগের প্রোটোকল সিদ্ধান্তগুলোও ব্যাখ্যা করে। Segment-ভিত্তিক HTTP streaming (HLS, DASH) বিনামূল্যে CDN caching পায় কারণ segment গুলো সাধারণ cacheable HTTP object — অন্যদিকে low-latency live streaming (WebRTC) এটি ত্যাগ করে UDP-এর পক্ষে, latency-র বিনিময়ে cacheability ছেড়ে দেয়।

### ৩. Transcoding একটি request নয়, একটি pipeline
**সমস্যা:** একটি আপলোড করা ভিডিওকে resolution, bitrate, এবং codec জুড়ে কয়েক ডজন rendition হতে হবে। এতে মিনিট থেকে ঘণ্টা সময় লাগে এবং এটি অত্যন্ত CPU-নিবিড়।

**যা শিখবেন:** এটি canonical asynchronous processing সমস্যা। Upload request ফাইলটি সংরক্ষণ করে এবং তাৎক্ষণিকভাবে ফেরত দেয়; একটি queue transcoding worker-দের একটি বহরকে (fleet) খাওয়ায়; ভিডিওর status processing থেকে ready-তে যায়। আপনি শিখবেন কীভাবে source-কে chunk-এ বিভক্ত করে, অনেক worker জুড়ে স্বাধীনভাবে transcode করে, এবং পুনরায় একত্রিত করে parallelize করতে হয় — যা একটি দুই ঘণ্টার serial job-কে কয়েক মিনিটে পরিণত করে। এবং আপনি শিখবেন এর মাঝখানে ব্যবহারকারী কী দেখে, কারণ "আপনার ভিডিও প্রসেস হচ্ছে" একটি প্রোডাক্ট state যা architecture তৈরি করেছে।

### ৪. Storage যা গুণিত হয়
**সমস্যা:** প্রতিটি ভিডিওর প্রতিটি rendition সংরক্ষণ করার মানে storage footprint চিরকালের জন্য মূল ফাইলের চেয়ে কয়েক গুণ বেশি।

**যা শিখবেন:** জনপ্রিয়তা অনুযায়ী Tiering — hot কনটেন্ট দ্রুতগতির storage-এ এবং edge-এ ব্যাপকভাবে replicate করা, cold কনটেন্ট সস্তা archival storage-এ ধীরগতির first-byte time মেনে নিয়ে। বিরল rendition গুলো eagerly-র বদলে lazily তৈরি করা। এবং estimation-এর অভ্যাস: প্রতি মিনিটে আপলোড হওয়া ঘণ্টা, গড় সাইজ দিয়ে গুণ, এবং rendition multiplier দিয়ে গুণ করলে এমন একটি সংখ্যা পাওয়া যায় যা ব্যবসার cost structure দৃশ্যমান করে তোলে।

### ৫. Metadata, recommendation, এবং যা কিছু বাইট নয়
**সমস্যা:** Search, recommendation, view count, comment, subscription, এবং watch history হলো সাধারণ অ্যাপ্লিকেশন সমস্যা যা একটি অসাধারণ delivery সমস্যার পাশাপাশি বিদ্যমান।

**যা শিখবেন:** control plane-কে data plane থেকে আলাদা করতে হবে, এবং প্রতিটিকে যথাযথভাবে আকার দিতে হবে। View count উচ্চ-আয়তনের এবং আনুমানিক (approximate) — প্রতিটি play-তে একটি synchronous database increment-এর চেয়ে asynchronous aggregation এবং probabilistic counting-এর জন্য নিখুঁত উপযুক্ত। Recommendation একটি batch বা streaming pipeline, request-time computation নয়। এটি চিনতে পারা যে এগুলো একটি প্রোডাক্টে *ভিন্ন সিস্টেম* যা একসাথে থাকে, সেটাই কাঠামোগত অন্তর্দৃষ্টি।

## ভুল করলে কী মূল্য দিতে হয়

- **CDN-কে পরবর্তী চিন্তা হিসেবে দেখা** এই ইন্টারভিউতে সবচেয়ে বড় ব্যর্থতা। এটি বাদ দেওয়া মানে এমন একটি architecture প্রস্তাব করা যা শারীরিকভাবে কাজ করতেই পারে না।
- **Synchronously Transcoding করা** আপলোডকে ঘণ্টার পর ঘণ্টা আটকে রাখে এবং CPU-bound কাজে request-handling ক্ষমতা অপচয় করে।
- **একটি একক bitrate সার্ভ করা** বিশ্বের বেশিরভাগ ব্যবহারকারীর জন্য নিয়মিত buffering তৈরি করে।
- **প্রতিটি play-তে database-এ একটি view counter increment করা** সবচেয়ে জনপ্রিয় কনটেন্টে একটি write hotspot তৈরি করে — যেখানে এটি সবচেয়ে বেশি ক্ষতি করে ঠিক সেখানেই।
- **Storage amplification উপেক্ষা করা** cost-কে বিশাল গুণকে কমিয়ে দেখায়।

## ইন্টারভিউয়াররা কেন এটি বেছে নেন

একজন প্রার্থী *আসল* constraint চিহ্নিত করতে পারে কিনা তার সেরা উপলব্ধ পরীক্ষা এটি। যে ব্যক্তি database schema দিয়ে শুরু করেন তিনি সমস্যাটি ভুল পড়েছেন; যিনি বলেন "এখানে প্রধান cost ও constraint হলো bandwidth, তাই CDN এবং encoding ladder ডিজাইনের মূল অংশ" তিনি এটি সঠিকভাবে পড়েছেন। এটি স্বাভাবিকভাবেই asynchronous pipeline, storage tiering, protocol selection, এবং এমন একটি স্কেলে capacity estimation-এর অনুশীলন করায় যেখানে সংখ্যাগুলো সত্যিকার অর্থেই গুরুত্বপূর্ণ।

## এটি কীভাবে সংযুক্ত

এই কেস স্টাডি core architecture হিসেবে **CDN** (topic 18)-এর উপর নির্মিত, streaming choice-এর জন্য **transport protocol** (topic 33), transcoding pipeline-এর জন্য **message queue** এবং worker pool (topic 20), analytics ও recommendation-এর জন্য **batch versus stream processing** (topic 23), **object storage এবং tiering** (topic 11), একাধিক স্তরে **caching** (topic 17), view এবং unique viewer-এর জন্য **probabilistic counting** (topic 42), **multi-region** distribution (topic 47), এবং একটি compute-heavy worker fleet-এর **horizontal scaling** (topic 4)।

**পরবর্তী:** [একটি Ride-Sharing সিস্টেম ডিজাইন করা](../54-design-a-ride-sharing-system-uber/why.md) — শেষ কেস স্টাডি, যেখানে geospatial matching এবং real-time state একত্রিত হয়।
</content>
