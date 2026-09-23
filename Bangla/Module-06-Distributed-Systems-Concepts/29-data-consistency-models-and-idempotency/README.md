# Distributed System-এ Data Consistency Models ও Idempotency

Difficulty: Advanced

## Learning Objectives

- Consistency model-এর spectrum ব্যাখ্যা করা — strong, causal, এবং eventual — এবং CAP/PACELC-এর সাপেক্ষে প্রতিটি কোথায় বসে তা বোঝা।
- Linearizability-কে নির্ভুলভাবে সংজ্ঞায়িত করা এবং এর performance ও availability cost বোঝা।
- Causal consistency এবং ব্যবহারিক মধ্যবর্তী guarantee-গুলো বোঝা (read-your-writes, monotonic reads, monotonic writes, session consistency)।
- Idempotency কী, নিরাপদ retry-এর জন্য এটি কেন অপরিহার্য, এবং একটি বাস্তব API-তে কীভাবে idempotency key implement করতে হয় তা সংজ্ঞায়িত করা।
- Module 6-এর concept-গুলো — hashing, rate limiting, circuit breaker, consensus, transaction, এবং consistency — কীভাবে একসাথে মিলে reliable distributed system তৈরির একটি সুসংগত toolkit গঠন করে তা দেখা।

## Script

### Hook / Intro

আমরা এই পুরো module জুড়ে আলোচনা করেছি কীভাবে সবকিছু ভুল হয়ে গেলেও distributed system-কে সচল রাখা যায় — কীভাবে load route করতে হয়, কীভাবে service-কে সুরক্ষিত রাখতে হয়, কীভাবে node-গুলোকে একমত করাতে হয়, কীভাবে service boundary জুড়ে transaction coordinate করতে হয়। আজ আমরা এমন একটি প্রশ্ন দিয়ে বৃত্তটি সম্পূর্ণ করব যা প্রায় সবকিছুর ভিত্তি: আপনার data যখন একাধিক node-এ থাকে, তখন client সেই data read করার সময় আসলে কী *প্রত্যাশা* করতে পারে? এটাই একটি consistency model সংজ্ঞায়িত করে। আর একবার বুঝে গেলে যে read এবং write একটি single machine-এর মতো ঠিক একই আচরণ নাও করতে পারে, তখন এটিকে বাস্তবে টিকে থাকার জন্য আমাদের একটি সহযোগী concept দরকার: idempotency, যা retry-কে — circuit breaker video-তে যা নিয়ে আমরা কথা বলেছিলাম — আসলেই নিরাপদে সম্পাদন করার উপায় করে দেয়।

### Consistency Models কেন গুরুত্বপূর্ণ

CAP এবং PACELC মনে করুন: আপনি যখন node-গুলো জুড়ে data replicate করেন, তখন আপনি partition-এর সময় এবং স্বাভাবিক operation-এর সময়, উভয় ক্ষেত্রেই consistency-কে availability এবং latency-এর বিপরীতে trade-off করছেন। একটি consistency model হলো সেই formal contract যা আপনাকে ঠিক বলে দেয় যে সেই trade-off-এর বিনিময়ে আপনি কী guarantee পাচ্ছেন। এটি কোনো বাইনারি "consistent বা not"-এর ব্যাপার নয় — এটি একটি spectrum, এবং আপনার use case-এর জন্য সেই spectrum-এ ভুল point বেছে নেওয়া distributed systems design-এর সবচেয়ে সাধারণ ভুলগুলোর একটি। Replication lag হলো এর গুরুত্বের পেছনের physical কারণ: যদি একটি write primary-তে পড়ে এবং asynchronously replica-গুলোতে propagate হয়, তাহলে এমন একটি window থাকে যেখানে ভিন্ন client যারা ভিন্ন replica read করছে তারা ভিন্ন উত্তর দেখতে পায়। Consistency model আপনাকে বলে দেয় সেই window-এর সময় কোন প্রতিশ্রুতিগুলো বহাল থাকে।

### Strong Consistency

সবচেয়ে শক্তিশালী উপযোগী guarantee হলো linearizability। সহজভাবে বলতে গেলে: প্রতিটি operation তার invocation এবং response-এর মধ্যবর্তী কোনো এক মুহূর্তে তাৎক্ষণিকভাবে কার্যকর হয় বলে মনে হয়, এবং সব client একই real-time order-এ operation দেখে — যেন data-র মাত্র একটিই copy আছে এবং প্রতিটি request একবারে একটি করে সেটির মধ্য দিয়ে গেছে। Linearizability ভালোভাবে compose হয় এবং সহজে যুক্তি করা যায়, যে কারণে এটি gold standard। কিন্তু এটি ব্যয়বহুল: এটি অর্জন করতে সাধারণত coordination দরকার হয় — একটি single leader, একটি quorum read/write protocol, অথবা Raft-এর মতো একটি consensus algorithm, যা সবই আমরা এই module-এ আগে কভার করেছি। Partition-এর সময়, একটি linearizable system-কে অবশ্যই minority side-এ থাকা request প্রত্যাখ্যান করতে হবে, stale বা conflicting data serve করার ঝুঁকি নেওয়ার বদলে — এটাই CAP-এর "C" কাজে লেগেছে। ZooKeeper, etcd, এবং Spanner (TrueTime এবং Paxos ব্যবহার করে)-এর মতো system linearizable read এবং write প্রদান করে কারণ leader election বা config data-র সঠিকতা latency cost-এর মূল্য দেয়।

### Eventual Consistency

অন্য প্রান্তে: eventual consistency শুধু প্রতিশ্রুতি দেয় যে যদি write বন্ধ হয়ে যায়, তাহলে সব replica *শেষপর্যন্ত* একই value-তে converge হবে। এটি ordering সম্পর্কে কিছু বলে না, এবং "শেষপর্যন্ত"-এ কতটা সময় লাগবে সে সম্পর্কেও কিছু বলে না। এটাই DNS আপনাকে দেয়, কিছু নির্দিষ্ট operation-এর জন্য S3 ঐতিহাসিকভাবে যা প্রদান করত, এবং Cassandra ও DynamoDB তাদের tunable consistency mode-এ default হিসেবে যা প্রদান করে। এর সুবিধা হলো বিশাল availability এবং কম latency — প্রতিটি replica স্বাধীনভাবে read serve করতে পারে এবং write গ্রহণ করতে পারে, এমনকি partition-এর সময়েও। খরচ হলো client stale data দেখতে পারে, এবং একই key-তে concurrent write-এর জন্য একটি conflict resolution strategy দরকার হয় — timestamp সহ last-write-wins, vector clocks, অথবা automatic merging-এর জন্য CRDTs।

### Causal Consistency

এই দুই চরমের মাঝখানে বসে আছে causal consistency, যা প্রায়শই বাস্তব application-এর জন্য sweet spot। এটি guarantee করে যে যদি operation A operation B-এর "happens-before" হয় — অর্থাৎ B, A-এর ফলাফল observe করেছে, যেমন একটি reply যা একটি comment-কে reference করে — তাহলে প্রতিটি node-কে অবশ্যই B-এর আগে A দেখতে হবে। যেসব operation-এর মধ্যে causal সম্পর্ক নেই সেগুলো ভিন্ন node-এ ভিন্ন order-এ দেখা যেতে পারে, যা ঠিক আছে কারণ তাদের relative order-এর উপর কিছু নির্ভর করে না। এটি ব্যবহারকারীদের প্রত্যাশিত অন্তর্দৃষ্টিমূলক সঠিকতার বেশিরভাগ দেয় — কেউ কোনো comment-এর reply, সেই comment-এর আগে দেখতে পায় না — global linearizability-র মূল্য না দিয়েই। Implementation-গুলো সাধারণত প্রতিটি write-এ সংযুক্ত vector clocks বা dependency metadata দিয়ে causality track করে।

### অন্যান্য Model সংক্ষেপে

বাস্তবে, system প্রায়ই client-centric guarantee প্রদান করে যা eventual consistency-এর কাছাকাছি বসে কিন্তু একটি single client-এর session-এর জন্য এর সবচেয়ে খারাপ surprise-গুলো ঠিক করে দেয়: read-your-writes (আপনি সবসময় আপনার নিজের আগের write দেখতে পাবেন), monotonic reads (পরপর read-এ আপনি কখনো সময়কে পেছনে যেতে দেখবেন না), এবং monotonic writes (আপনার write-গুলো যে order-এ আপনি issue করেছিলেন সেই order-এই apply হবে)। একসাথে বান্ডেল করা এগুলোকে প্রায়ই "session consistency" বলা হয়, এবং অনেক consumer-facing system — যেমন একটি social media timeline — আসলে এটাই implement করে, কারণ এটি একজন user থেকে সবচেয়ে অদ্ভুত eventual-consistency artifact লুকিয়ে রাখে, যদিও ভিন্ন ভিন্ন user global state-এর ভিন্ন ভিন্ন snapshot দেখতে পারে।

### Idempotency — এটি কেন গুরুত্বপূর্ণ

এবার, সহযোগী সমস্যা। Circuit breaker এবং retry video-তে, আমরা বলেছিলাম retry idempotent হতে হবে — এখন চলুন সেটা ঠিক কী মানে তা সংজ্ঞায়িত করি। একটি operation idempotent হয় যদি এটি একাধিকবার সম্পাদন করার প্রভাব একবার সম্পাদন করার মতোই হয়। একটি field-কে একটি fixed value-তে set করা idempotent; একটি counter-কে increment করা idempotent নয়। এটি distributed system-এ প্রচণ্ড গুরুত্বপূর্ণ কারণ network অস্পষ্ট উপায়ে fail করে — যদি একজন client একটি "charge $50" request পাঠায় এবং response পাওয়ার আগেই connection drop হয়ে যায়, তার কোনো ধারণা নেই যে server এটি process করেছে কিনা। যদি এটি retry করে এবং original request আসলেই সফল হয়ে থাকে, তাহলে একটি non-idempotent operation customer-কে দুইবার charge করে ফেলবে।

স্ট্যান্ডার্ড সমাধান হলো একটি idempotency key: client একটি logical operation-এর জন্য একটি unique ID তৈরি করে — retry-এর জন্য নয়, operation-এর জন্যই — এবং প্রতিটি attempt-এর সাথে সেটি পাঠায়। Server সেই ID দিয়ে key করা result persist করে রাখে, সাধারণত Redis-এর মতো একটি দ্রুত store-এ বা একটি unique constraint সহ একটি database table-এ। একই key দিয়ে retry হলে, server request reprocess করে না; এটি stored result খুঁজে বের করে ফেরত দেয়। এটাই ঠিক যেভাবে Stripe-এর payments API কাজ করে, এবং এই কারণেই HTTP specification অনুযায়ী PUT এবং DELETE-কে idempotent হিসেবে সংজ্ঞায়িত করে যেখানে POST নয় — দুটি identical POST দ্বারা একটি resource দুইবার তৈরি হওয়া একটি bug হবে, কিন্তু একটি PUT একই resource state দুইবার set করা নিরাপদ।

### Real-World Example

DynamoDB এবং Cosmos DB explicitly tunable consistency level প্রদান করে — আপনি প্রতি-request ভিত্তিতে "strongly consistent reads" (linearizable কিন্তু বেশি ব্যয়বহুল এবং সামান্য বেশি latency) এবং "eventually consistent reads" (সস্তা, দ্রুত, মাঝেমাঝে stale)-এর মধ্যে বেছে নিতে পারেন। Stripe money mutate করে এমন POST request-এ একটি `Idempotency-Key` header require করে, এবং response 24 ঘণ্টার জন্য cache করে রাখে যাতে একটি retried request দ্বিতীয় charge তৈরি করার বদলে identical result ফেরত দেয়। Kafka consumer-রা offset track করে এবং message ID-তে key করা upsert ব্যবহার করে idempotent processing implement করে, যাতে crash-এর পর একটি message reprocess করলে side effect duplicate না হয়।

### Recap

চলুন পুরো module-এর উপর একটু বড় পরিসরে দৃষ্টি দিই। Consistent hashing আমাদের cluster পরিবর্তন হলে ন্যূনতম বিঘ্ন সহ node জুড়ে data ও load distribute করতে দিয়েছে। Rate limiting service-কে overwhelmed হওয়া থেকে রক্ষা করেছে। Circuit breaker, retry, এবং bulkhead failure-কে cascade হওয়া থেকে আটকেছে, কিন্তু নিরাপদে retry হওয়ার জন্য operation-কে idempotent হতে হয়েছে — যা আমরা মাত্র নির্ভুলভাবে সংজ্ঞায়িত করলাম। Paxos ও Raft-এর মতো Consensus algorithm আমাদের একটি উপায় দিয়েছে যাতে failure সত্ত্বেও node-রা একমত হতে পারে, যা leader election এবং strongly consistent store উভয়ের ভিত্তি। Distributed transaction — 2PC এবং Saga — service জুড়ে কাজ coordinate করার দুটি খুবই ভিন্ন উপায় দেখিয়েছে, একটি atomicity-কে প্রাধান্য দেয়, অন্যটি availability-কে। আর আজ, consistency model আপনাকে ঠিক বলে দেয় সেই প্রতিটি পছন্দের সাথে আপনি কী guarantee কিনছেন। একসাথে, এই ছয়টি topic হলো সেই toolkit যা আপনি যখনই এমন কিছু design করতে বলা হবে যা একাধিক machine জুড়ে সঠিকভাবে চলতে হবে, তখন হাতে নেবেন।

### পরবর্তী কী

এতে Module 6-এর সমাপ্তি ঘটল — এই course-এর theoretical core। পরবর্তী, Module 7-এ, আমরা concept থেকে architecture-এ চলে যাব: আমরা শুরু করব classic monolith বনাম microservices বিতর্ক দিয়ে, এবং এখন পর্যন্ত আমরা যা তৈরি করেছি তা ব্যবহার করে যুক্তি দেব কখন কোনটা আসলে সঠিক হবে।

## Key Takeaways

- Consistency model-গুলো linearizability (strong, coordination-heavy) থেকে causal consistency হয়ে eventual consistency (weak, highly available) পর্যন্ত একটি spectrum গঠন করে।
- Linearizability প্রতিটি operation-কে তাৎক্ষণিক এবং সম্পূর্ণরূপে ordered বলে মনে করায়; এটি সাধারণত একটি leader বা consensus দরকার করে এবং partition-এর সময় availability বিসর্জন দেয়।
- Eventual consistency শুধুমাত্র convergence-এর guarantee দেয়, সময় বা ordering-এর কোনো সীমা ছাড়াই; এটি availability সর্বোচ্চ করে এবং concurrent write-এর জন্য একটি conflict-resolution strategy দরকার হয়।
- Causal consistency সম্পূর্ণ global ordering ছাড়াই happens-before ordering সংরক্ষণ করে, এবং প্রায়ই read-your-writes ও monotonic reads-এর মতো client-centric guarantee-এর সাথে জোড়া লাগানো হয়।
- Idempotency মানে একটি operation পুনরাবৃত্তি করার প্রভাব একবার করার মতোই; idempotency key client-কে side effect duplicate না করে অস্পষ্ট request নিরাপদে retry করতে দেয় (যেমন, double charge)।
- Consistency model-এর পছন্দ course-এর আগে করা CAP/PACELC trade-off-এর সরাসরি ফলাফল — এই module hashing, rate limiting, resilience pattern, consensus, এবং transaction-কে reliable distributed system-এর জন্য একটি একক সুসংগত framework-এ বেঁধে দেয়।
