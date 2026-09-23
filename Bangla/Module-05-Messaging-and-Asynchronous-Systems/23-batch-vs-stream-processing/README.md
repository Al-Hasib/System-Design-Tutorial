# Batch Processing বনাম Stream Processing

**Difficulty:** Intermediate/Advanced

## Learning Objectives

- Batch processing এবং stream processing সংজ্ঞায়িত করা এবং তাদের মূল পার্থক্যগুলো চিহ্নিত করা
- এই দুই মডেলের মধ্যে latency, throughput, এবং complexity-এর tradeoff বোঝা
- Stream processing-এ windowing ধারণাগুলো চেনা (tumbling, sliding, session windows)
- প্রতিটি পদ্ধতির জন্য সাধারণ tools চিহ্নিত করা (batch-এর জন্য Hadoop/Spark, stream-এর জন্য Kafka Streams/Flink/Spark Streaming)
- বাস্তব system design সিদ্ধান্তে Lambda এবং Kappa architecture ধারণা প্রয়োগ করা

## Script

### Hook/Intro

কাপড় কাচার দুটি উপায় কল্পনা করুন। প্রথম অপশন: আপনি অপেক্ষা করেন যতক্ষণ না বাস্কেট পুরো ভর্তি হয়, তারপর একবারে একটা বড় লোড চালান — efficient, মেশিন পুরো ক্ষমতায় ব্যবহার হয়, কিন্তু একটা পরিষ্কার শার্ট পেতে হয়তো এক সপ্তাহ অপেক্ষা করতে হতে পারে। দ্বিতীয় অপশন: প্রতিটা জিনিস নোংরা হওয়ার সাথে সাথে ধুয়ে ফেলেন, একটা একটা করে — আপনি সবসময় কয়েক মিনিটের মধ্যে একটা পরিষ্কার শার্ট পাবেন, কিন্তু মেশিন প্রায় সবসময় চলতে থাকে এবং প্রতি item-এ efficiency অনেক কম। এটাই batch processing এবং stream processing-এর মধ্যে পুরো tension, আর আজ আমরা ভেঙে দেখব কখন কোনটা সঠিক পছন্দ।

### Batch Processing কী?

Batch processing মানে একটা নির্দিষ্ট সময় ধরে — এক ঘণ্টা, একদিন, এক সপ্তাহ — ডেটা জমা করা এবং তারপর সেগুলো একসাথে একটা বড় job-এ প্রসেস করা। ভাবুন একটা রাতের job যা গতকালের সব transaction নিয়ে একটা financial report তৈরি করে, অথবা একটা সাপ্তাহিক job যা গত এক মাসের user behavior data ব্যবহার করে একটা machine learning model পুনরায় গণনা করে। এর মূল বৈশিষ্ট্য হলো, batch processing একটা bounded, finite dataset-এর উপর কাজ করে — এর একটা স্পষ্ট শুরু এবং শেষ থাকে, এবং job চলে, শেষ হয়, এবং একটা ফলাফল তৈরি করে।

Throughput-এর জন্য batch processing দারুণ। যেহেতু আপনি সবকিছু একসাথে প্রসেস করছেন, আপনি ব্যাপকভাবে optimize করতে পারেন — ডেটা sort করতে পারেন, efficient bulk algorithm ব্যবহার করতে পারেন, অনেকগুলো মেশিনে parallelize করতে পারেন, এবং প্রতি ইউনিট ডেটা প্রসেস করার জন্য সর্বোচ্চ efficiency আদায় করতে পারেন। ক্লাসিক উদাহরণ হলো Hadoop-এর MapReduce, অথবা আরও আধুনিক tool যেমন Apache Spark চালিয়ে batch job করা। Tradeoff হলো latency: পুরো batch শেষ না হওয়া পর্যন্ত ফলাফল পাওয়া যায় না। যদি আপনার batch রাতে একবার চলে, তাহলে সংজ্ঞানুসারেই আপনার ডেটা সবসময় অন্তত কয়েক ঘণ্টা পুরনো থাকে।

### Stream Processing কী?

Stream processing এটা উল্টে দেয়: batch জমা হওয়ার জন্য অপেক্ষা না করে, আপনি প্রতিটা ডেটা টুকরো — প্রতিটা event — যখনই আসে তখনই, ক্রমাগতভাবে প্রসেস করেন, dataset-এর কোনো সংজ্ঞায়িত "শেষ" ছাড়াই। ডেটাকে একটা unbounded, চিরপ্রবাহিত stream হিসেবে ধরা হয়, এবং আপনার processing logic প্রতিটা নতুন event-এ (অথবা events-এর micro-batch-এ) সেটা ঘটার মিলিসেকেন্ড থেকে সেকেন্ডের মধ্যে চলে।

এটাই real-time fraud detection-এর মতো জিনিসগুলোকে চালায় — একটা চুরি হওয়া credit card ফ্ল্যাগ করার জন্য আজ রাতের batch job-এর জন্য অপেক্ষা করা যায় না, swipe এবং approval-এর মধ্যকার দুই সেকেন্ডের মধ্যেই সেটা ধরতে হবে। অথবা "এই মুহূর্তে অনলাইনে থাকা users" দেখানো live dashboard। অথবা একটা server-এর error rate বেড়ে গেলে সাথে সাথে একটা alert trigger করা। এখানকার tools-এর মধ্যে আছে Kafka Streams, Apache Flink, এবং Spark Structured Streaming।

Tradeoff হলো complexity এবং, প্রায়ই, batch-এর তুলনায় প্রতি event-এ কম raw throughput efficiency, কারণ আপনি bulk-এ জিনিস প্রসেস করার সুবিধা পাচ্ছেন না। Stream processing systems-কে একটা সত্যিকারের কঠিন সমস্যার মুখোমুখিও হতে হয়: events সবসময় ক্রমানুসারে আসে না, এবং সবসময় সময়মতোও আসে না।

### Windowing: একটা অসীম Stream-কে বোঝা

যেহেতু একটা stream কখনো "শেষ" হয় না, "প্রতি মিনিটে গড় অর্ডার সংখ্যা"-র মতো কিছু আপনি কীভাবে গণনা করবেন? আপনাকে অসীম stream-কে finite chunk-এ কাটতে হবে, এবং একেই বলা হয় **windowing**। একটা **tumbling window** stream-কে fixed, non-overlapping chunk-এ ভাঙে — যেমন, প্রতি ৬০ সেকেন্ডে গণনা করো, তারপর নতুন করে শুরু করো। একটা **sliding window** overlap করে — উদাহরণস্বরূপ, "শেষ ৬০ সেকেন্ড", যা প্রতি ১০ সেকেন্ডে পুনরায় গণনা করা হয়, তাই windows-গুলো ডেটা শেয়ার করে। একটা **session window** activity-র মধ্যে ফাঁক অনুযায়ী events গ্রুপ করে — উদাহরণস্বরূপ, একজন user সক্রিয় থাকা পর্যন্ত তার click-গুলোকে একটা "session"-এ গ্রুপ করা, এবং, বলা যাক, ৩০ মিনিট নিষ্ক্রিয়তার পর window বন্ধ করা। সঠিক windowing strategy বেছে নেওয়া সঠিক real-time analytics ডিজাইনের কেন্দ্রবিন্দু।

এখানে **event time বনাম processing time**-এর একটা জটিল বিষয়ও আছে। Event time হলো কখন কিছু আসলে ঘটেছে; processing time হলো কখন আপনার system সেটা handle করতে পেরেছে। Network delay, retry, এবং out-of-order delivery-র কারণে এই দুটো ভিন্ন হয়ে যেতে পারে, এবং পরিশীলিত stream processor-গুলো watermarks-এর মতো concept ব্যবহার করে সিদ্ধান্ত নেয় "একটা window বন্ধ এবং তার ফলাফল চূড়ান্ত ধরার আগে late data-র জন্য কতক্ষণ অপেক্ষা করব।"

### Lambda এবং Kappa Architecture

ঐতিহাসিকভাবে, অনেক কোম্পানি দুটোই চেয়েছে: batch-এর accuracy এবং reprocessability, এবং streaming-এর low latency। এর ফলে তৈরি হয়েছে **Lambda architecture**: একই ডেটা দুটো parallel path-এ চালানো — একটা batch layer যা নিয়মিত accurate, comprehensive ফলাফল গণনা করে, এবং একটা speed layer যা দ্রুত, আনুমানিক real-time ফলাফল গণনা করে — তারপর query serve করার সময় দুটো view merge করা, batch layer-কে speed layer-এর যেকোনো drift শেষ পর্যন্ত সংশোধন করতে দেওয়া। এটা কাজ করে, কিন্তু আপনাকে একই ধরনের logic করা দুটো আলাদা codebase maintain করতে হয়, যা একটা সত্যিকারের operational burden।

**Kappa architecture** একটা সরলীকরণ হিসেবে প্রস্তাব করা হয়েছিল: সবকিছুকে একটা stream হিসেবে বিবেচনা করো, historical data সহ, এবং যখন কিছু পুনরায় গণনা করার প্রয়োজন হয় তখন সম্পূর্ণ আলাদা একটা batch pipeline maintain না করে stream-টাকে শুরু থেকে reprocess করো। এটা কেবল ব্যবহারিক কারণ Kafka-র মতো systems historical data-কে একটা log হিসেবে ধরে রাখতে এবং replay করতে পারে — যা, আশা করি, এই module-এর একদম প্রথম video থেকে পরিচিত মনে হবে। Kappa, Lambda-র কিছু flexibility-র বিনিময়ে একটা অনেক সহজ single-codebase system দেয়।

### একটা বাস্তব উদাহরণ

Spotify-এর মতো একটা কোম্পানির কথা ভাবুন। তাদের একই সাথে দুটো মডেলই প্রয়োজন। Batch processing আপনার "Discover Weekly" playlist তৈরি করে — এটা সপ্তাহে একবার চললেই ঠিক আছে, বিশাল historical listening data ভারী, ব্যয়বহুল machine learning model দিয়ে প্রসেস করে, কারণ সেই product-এর জন্য "সপ্তাহে একবার" freshness পুরোপুরি গ্রহণযোগ্য। কিন্তু stream processing হলো যা একজন artist-এর page-এ দেখা "currently playing" count-কে চালায়, অথবা একটা গান viral হওয়ার real-time detection যাতে সেটা তাৎক্ষণিকভাবে promote করা যায় — সেটা রাতের batch job-এর জন্য অপেক্ষা করতে পারে না; সেকেন্ডের মধ্যে প্রতিক্রিয়া দেখাতে হবে।

### Recap

চলুন গুটিয়ে নিই। Batch processing bounded, finite dataset-এর উপর কাজ করে, latency-র বিনিময়ে throughput এবং efficiency-কে optimize করে — periodic, বড়-স্কেল গণনার জন্য নিখুঁত যেখানে ঘণ্টা বা দিনের staleness গ্রহণযোগ্য। Stream processing unbounded, ক্রমাগত ডেটার উপর কাজ করে, প্রতিটা event low latency-তে প্রসেস করে, এবং windowing ও out-of-order/late data সামলাতে হয় — যখন আপনার সেকেন্ডের মধ্যে কিছুতে প্রতিক্রিয়া দেখানো প্রয়োজন তখন নিখুঁত। Lambda architecture দুটোরই সেরাটা পাওয়ার জন্য উভয়কে parallel-এ চালায়; Kappa architecture সবকিছুকে একটা replayable stream হিসেবে ধরে সরল করে। এই দুটোর মধ্যে বেছে নেওয়া কোনটা "ভালো" তা নিয়ে নয় — এটা আপনার use case-এর freshness requirement-কে সঠিক tool-এর সাথে মেলানোর ব্যাপার।

### এরপর কী

এর মাধ্যমে messaging এবং asynchronous systems-এর Module 5 শেষ হলো — আমরা queues-এর মৌলিক mechanics থেকে শুরু করে, pub-sub দিয়ে fan-out, event-driven architecture-এর বড়-পরিসরের philosophy, এবং এখন এসবের মধ্য দিয়ে প্রবাহিত ডেটা আপনি আসলে কীভাবে প্রসেস করবেন তা পর্যন্ত গিয়েছি। পরের module-এ, আমরা distributed systems ধারণায় গিয়ার পরিবর্তন করব — শুরু হবে consistent hashing দিয়ে, distributed systems design-এর সবচেয়ে elegant ধারণাগুলোর একটা। সেখানে দেখা হবে।

## Key Takeaways

- Batch processing একটা schedule অনুযায়ী bounded, finite datasets handle করে, latency-র বিনিময়ে (ঘণ্টা/দিনে মাপা data freshness) throughput optimize করে।
- Stream processing unbounded, ক্রমাগত ডেটা যেভাবে আসে সেভাবে handle করে, কিছুটা raw processing efficiency-র বিনিময়ে low latency (সেকেন্ড বা তার কম) optimize করে।
- Windowing (tumbling, sliding, session) হলো যেভাবে stream processing একটা অসীম stream-কে গণনাযোগ্য chunk-এ কাটে।
- Event time বনাম processing time এবং out-of-order/late data হলো stream processing-এর জন্য বিশেষভাবে core challenges।
- Lambda architecture accuracy এবং low latency-র জন্য parallel batch + speed layers চালায়; Kappa architecture একটা single replayable stream pipeline-এ সরল করে।
- সঠিক পছন্দ নির্ভর করে আপনার ফলাফল কতটা fresh হওয়া দরকার তার উপর, কোন approach স্বভাবগতভাবে "ভালো" তার উপর নয়।
