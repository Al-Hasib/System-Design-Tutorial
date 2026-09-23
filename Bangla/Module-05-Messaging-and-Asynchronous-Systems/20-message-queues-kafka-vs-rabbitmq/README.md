# Message Queues ব্যাখ্যা: Kafka vs RabbitMQ

**কঠিনতা:** Intermediate/Advanced

## Learning Objectives

- অ্যাসিনক্রোনাস, queue-based communication কেন সিঙ্ক্রোনাস কলের অসাধ্য সমস্যাগুলো সমাধান করে তা ব্যাখ্যা করা
- একটি message queue-এর মূল অ্যানাটমি বর্ণনা করা: producers, brokers, queues, consumers, acknowledgments
- Kafka এবং RabbitMQ-কে স্থাপত্যগতভাবে (log vs broker-and-queue) তুলনা করা এবং কখন কোনটি বেছে নিতে হবে তা জানা
- Delivery guarantees বোঝা: at-most-once, at-least-once, এবং exactly-once
- Backpressure, consumer groups, এবং partitioning-কে স্কেলিং প্রক্রিয়া হিসেবে চিহ্নিত করা

## Script

### Hook/Intro

কল্পনা করুন আপনি একটি কফি শপে আছেন। আপনি কাউন্টারে অর্ডার দেন, ক্যাশিয়ার আপনাকে একটি নম্বরসহ রসিদ দেয়, এবং আপনি গিয়ে বসে পড়েন। আপনার লাতে তৈরি না হওয়া পর্যন্ত আপনি কাউন্টারে দাঁড়িয়ে বারিস্তার দিকে তাকিয়ে থাকেন না। অর্ডারটি একটি queue-তে যায়, বারিস্তা একে একে সেগুলো সম্পন্ন করে, এবং কাজ শেষ হলে আপনার নম্বর ডাকা হয়। এটাই — এটাই একটি message queue। আজ আমরা আলোচনা করব কীভাবে এই সাধারণ ধারণাটি ইন্টারনেটের সবচেয়ে বড় কিছু সিস্টেমকে শক্তি যোগায়, এবং আমরা এটি করার জন্য দুটি সবচেয়ে বিখ্যাত টুল তুলনা করব: Apache Kafka এবং RabbitMQ।

এই ভিডিওর শেষে, আপনি ঠিক ঠিক জানবেন কেন "just call the other service directly" বড় স্কেলে ভেঙে পড়ে, এবং আপনি ইন্টারভিউতে গিয়ে আত্মবিশ্বাসের সাথে ব্যাখ্যা করতে পারবেন কখন Kafka আর কখন RabbitMQ বেছে নেবেন।

### সরাসরি কল কেন যথেষ্ট নয়?

সমস্যাটা দিয়ে শুরু করা যাক। ধরুন আপনার একটি e-commerce checkout service আছে। একটি অর্ডার দেওয়া হলে, আপনাকে করতে হবে: পেমেন্ট চার্জ করা, ইনভেন্টরি আপডেট করা, একটি কনফার্মেশন ইমেইল পাঠানো, শিপিং ওয়্যারহাউসকে জানানো, এবং analytics আপডেট করা। সাদামাটা পদ্ধতি হলো এই সবগুলো সিঙ্ক্রোনাসভাবে, একটার পর একটা, checkout request-এর ভেতরেই কল করা।

আজ যদি email service ধীরগতির হয় তাহলে কী হবে? আপনার checkout request আটকে যাবে। যদি maintenance-এর কারণে analytics service বন্ধ থাকে তাহলে কী হবে? আপনার checkout পুরোপুরি ব্যর্থ হবে, যদিও অর্ডার সফল হলো কিনা তার সাথে analytics-এর কোনো সম্পর্কই নেই। আপনি অজান্তেই আপনার পুরো checkout flow-এর availability-কে আপনার সবচেয়ে অবিশ্বস্ত dependency-র availability-র সাথে কাপল করে ফেলেছেন। এটা ভঙ্গুর, এবং এটাই সেই মূল সমস্যা যা asynchronous messaging সমাধান করে।

একটি message queue checkout service-কে একটাই কাজ করতে দেয় — একটি অর্ডার বসানো এবং "order placed" বলে একটি message publish করা — এবং সরে যেতে দেয়। ডাউনস্ট্রিম সার্ভিসগুলো তাদের নিজের গতিতে, নিজেদের সময়সূচি অনুযায়ী, যখনই প্রস্তুত হয় তখন সেই message তুলে নেয়। Producer এবং consumer সময়ের দিক থেকে decoupled: message পাঠানোর সময় consumer-এর চলমান থাকারও দরকার নেই। তারা স্থানের দিক থেকেও decoupled: producer-এর জানার দরকার নেই কতগুলো consumer আছে বা তারা কোথায় আছে। এবং তারা গতির দিক থেকেও decoupled: একটি ধীরগতির consumer producer-কে ধীর করে দেয় না।

### একটি Message Queue-এর অ্যানাটমি

ভেন্ডর যাই হোক না কেন, প্রতিটি message queue সিস্টেমেরই একই মৌলিক চরিত্র থাকে। একজন **producer** একটি message তৈরি করে এবং একটি **broker**-কে পাঠায় — মধ্যস্থতাকারী সার্ভার যেটি সাময়িকভাবে message সংরক্ষণ করে। Message-গুলো একটি **queue**-তে (অথবা Kafka-র ক্ষেত্রে, partitions দিয়ে গঠিত একটি **topic**-এ) থাকে যতক্ষণ না একজন **consumer** সেগুলো নিয়ে প্রসেস করে। প্রসেসিং শেষে, consumer broker-কে একটি **acknowledgment** পাঠায়, বলে "আমি শেষ করেছি, তুমি এটা মুছে দিতে পার" অথবা "আমি আমার read position কমিট করেছি।"

এই acknowledgment ধাপটি অত্যন্ত গুরুত্বপূর্ণ। প্রসেসিং চলাকালীন যদি কোনো consumer acknowledge না করেই ক্র্যাশ করে, তাহলে broker জানে যে message-টি অন্য একজন consumer-কে আবার পাঠাতে হবে। এভাবেই আলাদা আলাদা worker ব্যর্থ হলেও queue-গুলো নির্ভরযোগ্যতা অর্জন করে।

এখানে আরও দুটি ধারণা গুরুত্বপূর্ণ: **backpressure** এবং **consumer groups**। Backpressure ঘটে যখন producer-রা consumer-দের প্রসেস করার চেয়ে দ্রুত গতিতে message তৈরি করে — queue সেই burst-কে নিজের মধ্যে শুষে নেয়, যাতে ডাউনস্ট্রিম সিস্টেম ভেঙে না পড়ে। Consumer groups আপনাকে অনুভূমিকভাবে (horizontally) স্কেল করতে দেয়: একাধিক consumer instance একটি queue বা topic-এর কাজ ভাগ করে নেয়, প্রত্যেকে message-এর একটি উপসেট সামলায়, ফলে একটি সার্ভারের জন্য কাজ বেশি হয়ে গেলে আপনি আরও worker যোগ করতে পারেন।

### Kafka vs RabbitMQ: আর্কিটেকচারের পার্থক্য

এখন আসল তুলনায় আসা যাক, কারণ এখানেই মানুষ বিভ্রান্ত হয়, এবং এটা বোঝার জন্য সত্যিই সবচেয়ে গুরুত্বপূর্ণ পার্থক্য।

**RabbitMQ** হলো একটি প্রথাগত message broker যা queue-এর ধারণার উপর ভিত্তি করে তৈরি। একজন producer একটি "exchange"-এ message publish করে, যা নিয়ম অনুযায়ী সেটিকে এক বা একাধিক queue-তে রুট করে, এবং consumer-রা সেই queue থেকে message টেনে নেয়। একবার একটি message consume এবং acknowledge হয়ে গেলে, সেটা চলে যায় — queue থেকে মুছে যায়। RabbitMQ জটিল routing logic, per-message priority, এবং সূক্ষ্ম delivery guarantees-এ দুর্দান্ত। এটা task queue-এর জন্য দারুণ মানানসই — যেমন "process this image," "send this SMS," "run this background job" — যেখানে প্রতিটি message একটি স্বতন্ত্র কাজের একক প্রতিনিধিত্ব করে যা ঠিক একজন worker দ্বারা ঠিক একবার সম্পন্ন হওয়া উচিত।

**Kafka** সম্পূর্ণ ভিন্ন পদ্ধতি অবলম্বন করে। এটা প্রথাগত অর্থে আসলে একটা "queue" নয় — এটা একটি distributed, append-only log। Producer-রা একটি topic-এ message লেখে, যেটি স্কেলেবিলিটির জন্য partitions-এ ভাগ করা থাকে। Message পড়ার পরে সেগুলো মুছে যায় না; সেগুলো একটি নির্ধারিত retention period পর্যন্ত ডিস্কে টিকে থাকে, তা একদিন হোক বা চিরকাল। একাধিক consumer group ঠিক একই topic স্বাধীনভাবে পড়তে পারে, প্রত্যেকে log-এ নিজের read position ("offset") বজায় রাখে। এর অর্থ হলো Kafka একটি মেইলবক্সের চেয়ে বরং একটি DVR রেকর্ডিং-এর মতো — আপনি রিওয়াইন্ড করতে পারেন, রিপ্লে করতে পারেন, এবং একই সাথে ভিন্ন ভিন্ন পয়েন্ট থেকে দেখা পাঁচজন ভিন্ন দর্শক থাকতে পারে।

এই log-ভিত্তিক ডিজাইনের কারণেই Kafka বিশাল স্কেলে event-driven architecture এবং stream processing-এর ভিত্তি হয়ে উঠেছে — LinkedIn, Netflix এবং Uber-এর মতো কোম্পানিগুলো প্রতিদিন কোটি কোটি event সরাতে এটি ব্যবহার করে। RabbitMQ তখন উজ্জ্বল হয়ে ওঠে যখন আপনার সার্ভিসগুলোর মধ্যে স্মার্ট রাউটিং বা কম operational জটিলতাসহ ক্লাসিক task-queue semantics দরকার।

### Delivery Guarantees

প্রায় প্রতিটি সিস্টেম ইন্টারভিউতেই একটা প্রশ্ন আসবে: একটি message দুইবার প্রসেস হলে, বা একবারও না হলে কী হয়? জানার জন্য তিনটি delivery guarantee আছে। **At-most-once** মানে একটি message একবার পাঠানো হয় এবং কখনো পুনরায় চেষ্টা করা হয় না — যদি এটি হারিয়ে যায়, তাহলে চিরতরে হারিয়ে গেল; দ্রুত, কিন্তু ঝুঁকিপূর্ণ। **At-least-once** মানে broker একটি acknowledgment না পাওয়া পর্যন্ত পুনরায় চেষ্টা করতে থাকে, যা delivery নিশ্চিত করে কিন্তু প্রসেসিং সফল হওয়ার পরে ack-টি নিজেই হারিয়ে গেলে duplicate তৈরি করতে পারে। **Exactly-once** হলো পরম কাঙ্ক্ষিত অবস্থা — প্রতিটি message ঠিক একবার প্রসেস হয়, কোনো duplicate নেই, কোনো loss নেই — এবং সত্যিকার অর্থে distributed পরিবেশে এটি অর্জন করা কুখ্যাতভাবে কঠিন। Kafka তার নিজের ইকোসিস্টেমের মধ্যে idempotent producers এবং transactions-এর মাধ্যমে exactly-once semantics প্রদান করে, কিন্তু যেই মুহূর্তে আপনি একটি বহিরাগত সিস্টেমে প্রবেশ করেন, আপনাকে সাধারণত at-least-once delivery প্লাস idempotent consumers-এর জন্য ডিজাইন করতে হয় — অর্থাৎ আপনার প্রসেসিং লজিক একই ইনপুট দিয়ে দুইবার চালানো নিরাপদ হতে হবে।

### বাস্তব-জগতের উদাহরণ

Uber-এর কথা ভাবুন। একটি রাইড শেষ হলে, ডজনখানেক জিনিস ঘটতে হয়: রাইডারকে চার্জ করা, ড্রাইভারকে পেমেন্ট দেওয়া, ড্রাইভারের রেটিং আপডেট করা, fraud detection-এর জন্য ট্রিপ লগ করা, surge pricing মডেল আপডেট করা, এবং analytics ড্যাশবোর্ড ফিড করা। Uber এগুলোকে সিঙ্ক্রোনাসভাবে চেইন করে না। ট্রিপ-সমাপ্তির event একবার publish করা হয় — তাদের ক্ষেত্রে, Kafka-তে — এবং প্রতিটি ডাউনস্ট্রিম টিমের সার্ভিস স্বাধীনভাবে সাবস্ক্রাইব করে এবং প্রতিক্রিয়া দেখায়। Payments টিমের জানার দরকার নেই যে fraud টিম বলে কিছু আছে। যদি fraud detection service পুনরায় ডিপ্লয় করার কারণে সাময়িকভাবে অনুপলব্ধ থাকে, তবুও রাইড সম্পন্ন হয় এবং রাইডার তখনও চার্জ হয়, কারণ সেই message টেকসইভাবে সংরক্ষিত থাকে এবং fraud service অনলাইনে ফিরে এলে সেটা সেখানে অপেক্ষা করতে থাকবে।

### Recap

চলুন সংক্ষেপ করি। Message queue-গুলো producer এবং consumer-কে সময়, স্থান এবং গতির দিক থেকে decouple করে, আর এটাই distributed system-কে স্থিতিস্থাপক করে তোলে। প্রতিটি queue সিস্টেমই একই বিল্ডিং ব্লক শেয়ার করে: producers, brokers, queues বা topics, consumers, এবং acknowledgments। RabbitMQ হলো queue এবং স্মার্ট routing-কেন্দ্রিক একটি broker — task distribution-এর জন্য দারুণ। Kafka হলো একটি distributed, replayable log যা উচ্চ-থ্রুপুট event streaming-এর জন্য তৈরি — event-driven সিস্টেমের ভিত্তি হিসেবে দারুণ। আর delivery guarantee-গুলো at-most-once থেকে at-least-once হয়ে সেই অধরা exactly-once পর্যন্ত বিস্তৃত, যেখানে idempotency ব্যবহারিক ক্ষেত্রে আপনার সবচেয়ে বড় বন্ধু।

### এরপর কী

এখন যেহেতু আপনি message ইতস্তত সরানোর মৌলিক প্রক্রিয়া বুঝে গেছেন, এরপর আমরা এর উপর নির্মিত একটি নির্দিষ্ট এবং অবিশ্বাস্যভাবে শক্তিশালী প্যাটার্নে ফোকাস করব: publish-subscribe। আমরা দেখব কীভাবে একটি event পাবলিশারের না জেনে বা পাত্তা না দিয়েই ডজনখানেক স্বতন্ত্র subscriber-এর কাছে ছড়িয়ে (fan out) যেতে পারে। সেখানে দেখা হবে।

## Key Takeaways

- Asynchronous messaging producer এবং consumer-কে সময়, স্থান এবং গতির দিক থেকে decouple করে, যা সিঙ্ক্রোনাস চেইন থেকে হওয়া cascading failure এড়ায়।
- মূল উপাদানগুলো সব জায়গায় একই: producer, broker, queue/topic, consumer, acknowledgment।
- RabbitMQ = স্মার্ট routing-সহ প্রথাগত broker, task queue এবং per-message কাজ বণ্টনের জন্য আদর্শ।
- Kafka = distributed, replayable append-only log, উচ্চ-থ্রুপুট event streaming এবং একাধিক স্বতন্ত্র consumer-এর জন্য আদর্শ।
- Delivery guarantee-গুলো at-most-once থেকে at-least-once থেকে exactly-once পর্যন্ত বিস্তৃত; duplicate সামলানোর ব্যবহারিক উপায় হলো idempotent consumers।
- Consumer groups এবং partitioning হলো এই সিস্টেমগুলো অনুভূমিকভাবে স্কেল করার উপায়।
</content>
