# অনুশীলন ও Interview প্রশ্নসমূহ

**১. একটি Circuit Breaker-এর তিনটি state কী কী, এবং প্রতিটি transition কী ঘটায়?**
Closed (স্বাভাবিক, request গুলো পাস হয় এবং failure গণনা করা হয়), Open (failure rate/পরপর failure একটি threshold অতিক্রম করেছে, তাই সব call dependency-র সাথে যোগাযোগ না করেই সাথে সাথে ব্যর্থ হয়), এবং Half-Open (open-state timeout শেষ হয়ে গেছে, তাই সীমিত সংখ্যক probe request যেতে দেওয়া হয়)। Half-Open থেকে, যথেষ্ট সফল probe Closed-এ ফিরে যায়, আর যেকোনো/যথেষ্ট ব্যর্থ probe আবার Open-এ পাঠায়।

**২. কেন একটি Circuit Breaker স্বাভাবিকভাবে request timeout হতে দেওয়ার পরিবর্তে "দ্রুত ব্যর্থ" হয়?**
প্রতিটি request-কে তার পূর্ণ timeout পর্যন্ত চালাতে দিলে thread, connection, এবং queue slot এমন একটি call-এ আটকে থাকে যা যাইহোক ব্যর্থ হওয়ার সম্ভাবনা বেশি। দ্রুত ব্যর্থ হওয়া সাথে সাথে সেই resource গুলোকে অন্য কাজের জন্য মুক্ত করে দেয়, যা ঠিক caller-এর নিজের resource pool শেষ হয়ে failure উপরের দিকে cascade হওয়া প্রতিরোধ করে।

**৩. একটি outage-এর সময় naive retry-on-failure কেন বিপজ্জনক?**
যদি একটি dependency ইতিমধ্যে সমস্যায় থাকে এবং প্রতিটি client ব্যর্থতার সাথে সাথে retry করে, retry গুলো একটি ইতিমধ্যে overloaded system-এর উপর আরও load যোগ করে, outage-কে আরও খারাপ করে তোলে — এটাই thundering herd প্রভাব। Backoff ছাড়া, একটি retry storm একটি আংশিক degradation-কে একটি সম্পূর্ণ outage-এ পরিণত করতে পারে।

**৪. Jitter সহ exponential backoff ব্যাখ্যা করুন এবং কেন jitter গুরুত্বপূর্ণ।**
Exponential backoff একটি সমস্যাগ্রস্ত dependency-কে পুনরুদ্ধারের সময় দেওয়ার জন্য retry চেষ্টার মধ্যে delay exponentially বাড়ায় (যেমন, ১০০ms, ২০০ms, ৪০০ms, ৮০০ms)। Jitter প্রতিটি delay-তে randomness যোগ করে যাতে একই মুহূর্তে ব্যর্থ হওয়া অনেক client আবার ঠিক একই মুহূর্তে retry না করে — jitter ছাড়া, শুধু backoff তবুও synchronized retry-এর wave তৈরি করতে পারে।

**৫. কেন retry-কে শুধুমাত্র idempotent অপারেশনে সীমাবদ্ধ থাকতে হবে, নাহলে অন্যভাবে সুরক্ষিত থাকতে হবে?**
একটি idempotent অপারেশন যতবারই প্রয়োগ করা হোক না কেন একই ফলাফল দেয়, তাই এটি retry করা নিরাপদ (যেমন, একটি GET বা একটি PUT যা একটি absolute value সেট করে)। একটি non-idempotent অপারেশন, যেমন "এই credit card-এ charge করুন" বা "এই counter বাড়ান", অন্ধভাবে retry করলে duplicate side effect ঘটাতে পারে — এটি সাধারণত client-কে একটি idempotency key পাঠাতে দিয়ে সমাধান করা হয় যা server পুনরাবৃত্ত request শনাক্ত করতে এবং নিরাপদে বাদ দিতে ব্যবহার করে।

**৬. Retry budget কী, এবং এটি কেন উপযোগী?**
একটি retry budget অনুমোদিত retry-এর সংখ্যা বা percentage সীমাবদ্ধ করে (যেমন, retry মোট request volume-এর ১০%-এর বেশি হতে পারবে না)। এটি একটি ব্যাপক outage-এর সময় retry-কে নীরবে traffic ফুলিয়ে ফেলা থেকে বিরত রাখে, সবচেয়ে খারাপ failure পরিস্থিতিতেও system-এর সামগ্রিক load সীমাবদ্ধ রাখে।

**৭. Bulkhead প্যাটার্ন কী, এবং এটি কোন বাস্তব-জগতের ধারণা থেকে নামকরণ করা হয়েছে?**
Bulkhead প্যাটার্ন প্রতিটি dependency বা workload-এর জন্য resource (thread pool, connection pool, semaphore) আলাদা করে যাতে একটির exhaustion বাকিগুলোকে অনাহারে না রাখে। এটি একটি জাহাজের watertight compartment (bulkhead) থেকে নামকরণ করা হয়েছে, যা পুরো জাহাজকে ডুবিয়ে দেওয়ার পরিবর্তে বন্যাকে একটি অংশে সীমাবদ্ধ রাখে।

**৮. Bulkhead isolation বাস্তবায়নের তিনটি সুনির্দিষ্ট উপায়ের নাম বলুন।**
(১) প্রতিটি downstream dependency-র জন্য dedicated thread pool, (২) প্রতিটি downstream target-এর জন্য আলাদা, সীমিত connection pool, (৩) একটি সম্পূর্ণ thread pool ছাড়া হালকা isolation-এর জন্য semaphore-ভিত্তিক concurrency limit, এবং/অথবা infrastructure স্তরে tenant/workload আলাদা করতে আলাদা service instance বা node pool।

**৯. একটি downstream payment service ধীর কিন্তু সম্পূর্ণভাবে down নয়। আপনি যে retry + circuit breaker আচরণ চান তা ডিজাইন করুন।**
Payment call-টিকে একটি bulkhead-এ (dedicated thread pool/semaphore) wrap করুন যাতে এর ধীরগতি অন্য জায়গায় প্রয়োজনীয় thread শেষ করতে না পারে। এটিকে একটি circuit breaker-এ wrap করুন যা একটি rolling window জুড়ে failure/timeout rate track করে; threshold trip হওয়ার সাথে সাথে, breaker open হয় এবং একটি সমস্যাগ্রস্ত service-এর বিরুদ্ধে request queue করতে থাকার পরিবর্তে দ্রুত ব্যর্থ হয়। Retry-এর জন্য jitter সহ exponential backoff এবং একটি ছোট সীমিত attempt count ব্যবহার করা উচিত, শুধুমাত্র breaker Closed বা Half-Open থাকা অবস্থায় প্রয়োগ করা (কখনো একটি Open breaker-এর বিরুদ্ধে retry করবেন না), এবং শুধুমাত্র payment call-এর নিরাপদে idempotent অংশগুলোতে (যেমন, একটি idempotency key দ্বারা সুরক্ষিত), একজন customer-কে দ্বিগুণ charge করা এড়াতে।

**১০. আপনি যদি Circuit Breaker-এর open-state timeout খুব ছোট সেট করেন তাহলে কী হয়? খুব দীর্ঘ হলে?**
খুব ছোট হলে, breaker dependency প্রকৃতপক্ষে পুনরুদ্ধার হওয়ার আগেই Half-Open-এ চলে যায় এবং traffic পুনরায় প্রবেশ করতে দেয়, যার ফলে এটি বারবার আবার Open-এ ফিরে যায় ("flapping") এবং dependency-কে কখনো প্রকৃত শ্বাস নেওয়ার জায়গা দেয় না। খুব দীর্ঘ হলে, breaker একটি ইতিমধ্যে পুনরুদ্ধার হওয়া dependency-তে দ্রুত ব্যর্থ হতে থাকে, অপ্রয়োজনীয়ভাবে functionality খারাপ করে এবং সম্পূর্ণ service পুনরুদ্ধার বিলম্বিত করে।

**১১. একটি একক resilient call path-এ Circuit Breaker, Retry, এবং Bulkhead কীভাবে একসাথে composed হয়?**
Bulkhead প্রথমে প্রয়োগ করা হয় concurrency সীমাবদ্ধ করতে এবং সেই dependency-র জন্য resource আলাদা করতে; circuit breaker call-টিকে wrap করে ধারাবাহিক failure শনাক্ত করতে এবং trip হওয়ার সাথে সাথে দ্রুত ব্যর্থ হতে; retry logic সবচেয়ে বাইরের স্তরটি wrap করে, ক্ষণস্থায়ী সমস্যা সামলাতে backoff এবং jitter ব্যবহার করে — কিন্তু এটি শুধুমাত্র circuit Closed বা Half-Open থাকা অবস্থায় চালু হওয়া উচিত, কারণ একটি Open breaker-এর বিরুদ্ধে retry করা এটিকে trip করানোর উদ্দেশ্যকেই ব্যর্থ করে দেয়।

**১২. এই প্যাটার্নগুলো বাস্তবায়নকারী দুটি বাস্তব-জগতের system/tool দিন, এবং যেকোনো গুরুত্বপূর্ণ সতর্কতা উল্লেখ করুন।**
Resilience4j JVM-এর জন্য circuit breaker, retry, এবং bulkhead-কে composable decorator হিসেবে বাস্তবায়ন করে, এবং এটি Netflix-এর Hystrix-এর ব্যাপকভাবে সুপারিশকৃত উত্তরসূরি, যা এখন maintenance mode-এ আছে। Istio outlier detection এবং connection pool limit-এর মাধ্যমে service mesh স্তরে declaratively circuit breaking বাস্তবায়ন করে, যা application code পরিবর্তন ছাড়াই sidecar proxy দ্বারা কার্যকর করা হয়; AWS SDK গুলোও throttling-এর মতো retryable error-এর জন্য ডিফল্টভাবে jitter সহ exponential backoff বাস্তবায়ন করে।
