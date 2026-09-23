# Circuit Breaker, Retry & Bulkhead Patterns

কঠিনতা: Advanced

## শেখার লক্ষ্যসমূহ

- একটি distributed system-এ cascading failure কীভাবে ছড়িয়ে পড়ে এবং naive fault handling কেন সেটাকে আরও খারাপ করে তোলে তা ব্যাখ্যা করা।
- Circuit Breaker প্যাটার্নের তিনটি state (Closed, Open, Half-Open), যে thresholds গুলো transition ঘটায়, এবং এটি কীভাবে একটি সমস্যাগ্রস্ত dependency-কে রক্ষা করে তা বর্ণনা করা।
- exponential backoff এবং jitter ব্যবহার করে একটি সঠিক Retry কৌশল ডিজাইন করা, এবং কেন retry শুধুমাত্র idempotent অপারেশনে নিরাপদ তা ব্যাখ্যা করা।
- dedicated thread pool, connection pool, বা semaphore-এর মাধ্যমে failure domain গুলোকে আলাদা করতে Bulkhead প্যাটার্ন প্রয়োগ করা।
- Circuit Breaker, Retry, এবং Bulkhead-কে একটি একক resilient call path-এ একত্রিত করা এবং বাস্তব-জগতের implementation গুলো চিনতে পারা।

## স্ক্রিপ্ট

### Hook / ভূমিকা

কল্পনা করুন, একটি ধীরগতির database query আপনার পুরো platform-কে ফেলে দিচ্ছে — database crash হওয়ার কারণে নয়, বরং কারণ আপনার web server-এর প্রতিটি request thread সেটির জন্য অপেক্ষা করে আটকে আছে, আর এখন কেউ homepage পর্যন্ত serve করতে পারছে না। এটা কোনো hypothetical নয়। এটাই cascading failure, এবং প্রোডাকশনে distributed system গুলো ধ্বংস হওয়ার সবচেয়ে সাধারণ উপায়গুলোর একটি। আজ আমরা এমন তিনটি প্যাটার্ন নিয়ে আলোচনা করব যা এটি ঘটতে বাধা দেয়: Circuit Breaker, Retry, এবং Bulkhead। এগুলো distributed system-এর seatbelt এবং airbag-এর মতো — আপনি আশা করেন যেন কখনো এগুলোর প্রয়োজন না হয়, কিন্তু যেদিন হবে, সেদিন একটি ছোট্ট সমস্যা আর সম্পূর্ণ outage-এর মধ্যে একমাত্র প্রতিরোধকারী হবে এগুলোই।

### Cascading Failures — সমস্যাটি

সাধারণত এটি এভাবে শুরু হয়। আপনার কাছে আছে Service A, যা Service B-কে কল করছে, যা আবার একটি downstream Service C-কে কল করছে। Service C ধীরে সাড়া দিতে শুরু করে — হয়তো এটি load-এ আছে, হয়তো একটি disk ব্যর্থ হচ্ছে, হয়তো একটি network link degraded। এটি সম্পূর্ণভাবে down নয়, যা আসলে সবচেয়ে খারাপ ঘটনা, কারণ একটি hard failure দ্রুত ব্যর্থ হয়। বরং একটি ধীরগতির dependency resource আটকে রাখে।

Service B-এর thread গুলো C-এর জন্য অপেক্ষা করতে করতে block হতে শুরু করে। এর thread pool ভরে যায়। এখন B কোনো request-ই serve করতে পারে না — এমনকি যেগুলো C-কে স্পর্শও করে না — কারণ কোনো thread অবশিষ্ট নেই। Service A দেখে B timeout হচ্ছে, আর যদি A কোনো সংযম ছাড়া আগ্রাসীভাবে retry করে, তাহলে এটি ইতিমধ্যে সমস্যাগ্রস্ত B এবং C-এর উপর আরও বেশি load যোগ করে। এটাই thundering herd problem যা একটি failure-কে আরও তীব্র করে তোলে। কয়েক সেকেন্ডের মধ্যেই, একটি নিম্ন-স্তরের dependency-তে একটি ধীরগতি call stack-এর কয়েক স্তর উপরে সম্পূর্ণ unavailability-তে পরিণত হয়। এটাই cascading failure, আর এই তিনটি প্যাটার্ন ঠিক এটাই প্রতিরোধ করার জন্য বিদ্যমান।

### Circuit Breaker প্যাটার্ন

Circuit Breaker প্যাটার্নটি সরাসরি electrical engineering থেকে ধার করা — একটি physical breaker fault হলে current flow বন্ধ করার জন্য trip করে, বাকি circuit-কে রক্ষা করে। software-এ, এটি একটি remote dependency-তে করা call-কে wrap করে এবং এর success এবং failure rate track করে।

তিনটি state রয়েছে। প্রথমটি হলো **Closed** — স্বাভাবিক state। request গুলো যথারীতি dependency-তে প্রবাহিত হয়, এবং breaker শুধু failure গণনা করে, সাধারণত একটি rolling window জুড়ে, যেমন "গত ২০টি request" বা "গত ১০ সেকেন্ড।" যদি failure rate একটি নির্ধারিত threshold অতিক্রম করে — ধরুন, ৫০%-এর বেশি request ব্যর্থ হলে, বা পাঁচটি পরপর timeout হলে — breaker **Open** state-এ trip করে।

Open state-এ, breaker dependency-কে কল করা সম্পূর্ণভাবে বন্ধ করে দেয়। প্রতিটি call সাথে সাথে ব্যর্থ হয় — দ্রুত, সস্তা, in-process — timeout-এর জন্য অপেক্ষা করার পরিবর্তে। এটাই মূল অন্তর্দৃষ্টি: দ্রুত ব্যর্থ হওয়া thread এবং connection মুক্ত করে দেয়, সেগুলোকে জমা হতে না দিয়ে। breaker একটি নির্ধারিত timeout সময়ের জন্য Open থাকে, যাকে প্রায়ই "sleep window" বলা হয়, হয়তো ৩০ সেকেন্ড বা এক মিনিট।

সেই timeout শেষ হওয়ার পর, breaker **Half-Open**-এ চলে যায়। এটি একটি probing state — এটি অল্প কিছু test request পাঠায় দেখার জন্য dependency পুনরুদ্ধার হয়েছে কিনা। যদি সেই test call গুলো সফল হয় — একটি নির্দিষ্ট success threshold পূরণ করলে, যেমন ৩-এর মধ্যে ৩টি, বা একটি নির্দিষ্ট success percentage — breaker আবার Closed-এ ফিরে যায় এবং স্বাভাবিক traffic পুনরায় শুরু হয়। যদি সেগুলো ব্যর্থ হয়, এটি সাথে সাথে আবার Open-এ ফিরে যায় এবং আবার probe করার আগে আরেকটি পূর্ণ timeout সময়ের জন্য অপেক্ষা করে। এই half-open probing-ই breaker-কে সদ্য-পুনরুদ্ধারকারী একটি service-এর উপর তার timeout শেষ হওয়ার সাথে সাথে পুরো traffic দিয়ে আঘাত করা থেকে বিরত রাখে।

এখানে যে tuning knob গুলো গুরুত্বপূর্ণ সেগুলো হলো: failure-rate threshold, সেই rate গণনা করতে ব্যবহৃত rolling window-এর আকার, open-state timeout-এর সময়কাল, এবং half-open probe request-এর সংখ্যা ও success criteria। এগুলো ভুল সেট করলে — অতিরিক্ত sensitive হলে আপনি ছোটখাটো সমস্যাতেও trip করবেন; খুব শিথিল হলে, আপনি কিছুই রক্ষা করবেন না।

### Retry প্যাটার্ন

Retry দেখতে সহজ মনে হয় — একটি call ব্যর্থ হয়েছে, আবার চেষ্টা করুন — কিন্তু naive retry আসলে বিপজ্জনক, বিশেষ করে একটি outage-এর সময়। যদি একটি service সমস্যায় থাকে এবং প্রতিটি client ব্যর্থতার সাথে সাথে retry করে, তাহলে আপনি একটি ইতিমধ্যে ব্যর্থ system-এর উপর load কয়েকগুণ বাড়িয়ে দিলেন। এই কারণেই retry logic-এর প্রকৃত গঠন প্রয়োজন।

স্ট্যান্ডার্ড পদ্ধতি হলো **exponential backoff**: সাথে সাথে retry করার পরিবর্তে, আপনি প্রতিটি চেষ্টার মধ্যে ক্রমবর্ধমানভাবে বেশি সময় অপেক্ষা করেন — ধরুন ১০০ms, তারপর ২০০ms, তারপর ৪০০ms, তারপর ৮০০ms, প্রতিবার দ্বিগুণ হয়ে একটি নির্দিষ্ট সর্বোচ্চ সীমা পর্যন্ত। এটি downstream service-কে ক্রমাগত আঘাত পাওয়ার পরিবর্তে পুনরুদ্ধারের জন্য শ্বাস নেওয়ার জায়গা দেয়।

কিন্তু শুধু exponential backoff-এরও একটি ত্রুটি আছে: যদি হাজার হাজার client একই মুহূর্তে ব্যর্থ হয় — ধরুন, একটি সংক্ষিপ্ত network blip-এর কারণে — তাহলে তারা সবাই ঠিক একই সময়সূচি অনুযায়ী back off করবে এবং সবাই আবার ঠিক একই মুহূর্তে retry করবে। এটাকে thundering herd problem বলা হয়, এবং এটি **jitter** দিয়ে সমাধান করা হয় — প্রতিটি backoff interval-এ randomness যোগ করা যাতে retry গুলো synchronized wave-এ আসার বদলে সময় জুড়ে ছড়িয়ে যায়। AWS-এর সুপরিচিত "full jitter" পদ্ধতি প্রতিটি চেষ্টার জন্য শূন্য এবং গণনাকৃত exponential backoff value-এর মধ্যে একটি random delay বেছে নেয়।

আপনার একটি **retry budget**-ও দরকার — মোট retry-এর উপর একটি সীমা, হয় একটি absolute count হিসেবে অথবা সামগ্রিক request volume-এর একটি percentage হিসেবে, যাতে একটি খারাপ outage-এর সময় retry গুলো কখনো আপনার traffic-এর সংখ্যাগরিষ্ঠ অংশ হয়ে না যায়। এবং সবচেয়ে গুরুত্বপূর্ণ: retry শুধুমাত্র **idempotent** অপারেশনের ক্ষেত্রে নিরাপদ — এমন একটি অপারেশন যা যতবারই প্রয়োগ করা হোক না কেন একই ফলাফল দেয়। একটি GET request retry করা নিরাপদ। একটি idempotency key ছাড়া "এই credit card-এ charge করুন" POST retry করা একজন customer-কে দ্বিগুণ charge করে ফেলতে পারে। এই কারণেই বাস্তব-জগতের retry system গুলো প্রায় সবসময় server side-এ idempotency key-এর সাথে জোড়া লাগানো থাকে, যাতে একটি duplicate request চেনা যায় এবং পুনরায় প্রয়োগ না করে নিরাপদে বাদ দেওয়া যায়।

### Bulkhead প্যাটার্ন

Bulkhead প্যাটার্নটি তার নাম নিয়েছে ship design থেকে — একটি জাহাজের hull কে watertight compartment-এ ভাগ করা হয়, যাতে একটি compartment প্লাবিত হলে, পানি সেখানেই আটকে থাকে এবং পুরো জাহাজকে ডুবিয়ে দেয় না। software-এ একই ধারণা প্রয়োগ করুন: প্রতিটি dependency-র জন্য resource আলাদা করুন যাতে একটি ব্যর্থ dependency অন্য সবকিছুর জন্য প্রয়োজনীয় resource শেষ করতে না পারে।

স্পষ্টভাবে বললে, এর মানে হলো প্রতিটি downstream dependency-কে তার নিজস্ব dedicated thread pool, connection pool, বা semaphore দেওয়া, সব call-এর মধ্যে একটি সাধারণ pool ভাগ করার পরিবর্তে। আমাদের আগের cascading failure দৃশ্যটি মনে আছে? যদি Service B ধীরগতির Service C-তে call করার জন্য একটি আলাদা, সীমিত thread pool ব্যবহার করত — ধরুন সর্বোচ্চ ১০টি thread-এ সীমাবদ্ধ — তাহলে সেই ১০টি thread-ই C-এর জন্য অপেক্ষা করতে করতে আটকে গেলেও, B-এর বাকি thread pool অন্য সব ধরনের request serve করার জন্য মুক্ত থাকত। C-এর ধীরগতির blast radius শুধুমাত্র C-কে কল করা code path-এই সীমাবদ্ধ থাকে।

Bulkhead একাধিক স্তরে বাস্তবায়ন করা যায়: thread pool isolation (প্রতিটি dependency তার নিজস্ব executor পায়), semaphore isolation (একটি হালকা counter যা একটি সম্পূর্ণ pool ছাড়াই concurrent call সীমিত করে, খুব বেশি volume-এর call-এর জন্য উপযোগী), connection pool isolation (একটি database বা HTTP client প্রতিটি downstream target-এর জন্য একটি সীমিত, dedicated connection pool পায়), এবং এমনকি infrastructure স্তরেও — বিভিন্ন workload class-এর জন্য আলাদা service instance বা node pool, যাতে একটি noisy বা অসদাচরণকারী tenant পুরো shared capacity গ্রাস করতে না পারে।

### প্যাটার্নগুলোকে একত্রিত করা

এই তিনটি প্যাটার্ন পরস্পর পরিপূরক, প্রতিযোগী নয় — production system গুলো এদের একসাথে ব্যবহার করে। একটি সাধারণ resilient call প্রথমে একটি **bulkhead**-এ wrap করে concurrency সীমিত করতে এবং resource আলাদা করতে, তারপর একটি **circuit breaker**-এ wrap করে ধারাবাহিক failure শনাক্ত করতে এবং দ্রুত ব্যর্থ হতে, আর retry logic পুরো জিনিসটির চারপাশে থাকে, কিন্তু শুধুমাত্র তখনই retry করে যখন breaker Closed বা Half-Open থাকে — কখনোই একটি Open breaker-এর বিরুদ্ধে retry করে না, কারণ তা করলে এটিকে প্রথমে trip করানোর উদ্দেশ্যই ব্যর্থ হয়ে যায়। ক্রম গুরুত্বপূর্ণ: bulkhead concurrency সীমাবদ্ধ করে, circuit breaker একটি পরিচিত-খারাপ dependency-তে অপচয়িত call প্রতিরোধ করে, এবং backoff ও jitter সহ retry অবশিষ্ট ক্ষণস্থায়ী, পুনরুদ্ধারযোগ্য সমস্যাগুলো সামলায়।

### বাস্তব-জগতের উদাহরণ

আপনাকে এগুলো শুরু থেকে তৈরি করতে হবে না। Netflix এই চিন্তাধারার অনেকটাই পথপ্রদর্শক ছিল তাদের **Hystrix**-এর মাধ্যমে, তাদের circuit breaker library যা microservices জগতে এই প্যাটার্নকে জনপ্রিয় করেছিল — Hystrix এখন maintenance mode-এ আছে, এবং এর ব্যাপকভাবে সুপারিশকৃত উত্তরসূরি হলো **resilience4j**, একটি হালকা Java library যা circuit breaker, retry, bulkhead, এবং rate limiter composable decorator হিসেবে প্রদান করে। JVM জগতের বাইরে, **AWS SDK** গুলো throttling response-এর মতো retryable error-এর জন্য ডিফল্টভাবে jitter সহ exponential backoff বাস্তবায়ন করে, তাই AWS service কল করার সময় আপনি প্রথম থেকেই সঠিক retry behavior পান। এবং infrastructure স্তরে, Istio-এর মতো service mesh গুলো declaratively circuit breaking বাস্তবায়ন করে — আপনি একটি Kubernetes service-এ outlier detection এবং connection pool limit configure করেন, এবং mesh-এর sidecar proxy গুলো আপনার application code-কে এ সম্পর্কে কিছু জানার প্রয়োজন ছাড়াই অসুস্থ endpoint গুলোর ejection কার্যকর করে।

### সংক্ষিপ্তকরণ

চলুন সবকিছু একসাথে বাঁধি। Cascading failure ঘটে যখন একটি ধীরগতির বা ব্যর্থ dependency call chain-এর উপরে shared resource শেষ করে ফেলে। Circuit Breaker প্যাটার্ন একটি failure threshold trip হওয়ার সাথে সাথে দ্রুত ব্যর্থ হয়ে এটি বন্ধ করে, একটি Open state-এ প্রবেশ করে, তারপর সম্পূর্ণভাবে আবার Closed হওয়ার আগে Half-Open state-এর মাধ্যমে সতর্কতার সাথে পুনরুদ্ধার probe করে। Retry প্যাটার্ন jitter সহ exponential backoff এবং একটি retry budget ব্যবহার করে ক্ষণস্থায়ী failure নিরাপদে সামলায়, কিন্তু শুধুমাত্র idempotent অপারেশনে। এবং Bulkhead প্যাটার্ন প্রতিটি dependency-র জন্য resource pool আলাদা করে, জাহাজের watertight compartment-এর মতো, যাতে একটি dependency-র ব্যর্থতা বাকি সবকিছুকে ডুবিয়ে দিতে না পারে। একসাথে, এগুলো এমন system তৈরির মূল toolkit যা সম্পূর্ণভাবে ভেঙে পড়ার পরিবর্তে সুন্দরভাবে degrade হয়।

### এরপর কী

এরপরে, আমরা distributed systems theory-এর গভীরে যাচ্ছি: Consensus Algorithms — Paxos এবং Raft। আমরা দেখব কীভাবে distributed node গুলো একটি একক সত্যের উৎসে সম্মত হয় এমনকি যখন তাদের কিছু ব্যর্থ হয়, যা distributed database থেকে শুরু করে Kubernetes-এর leader election পর্যন্ত সবকিছুর ভিত্তি। সেখানে দেখা হবে।

## মূল বিষয়সমূহ

- Cascading failure ছড়ায় কারণ একটি ধীরগতির dependency call chain-এর উপর পর্যন্ত shared resource (thread, connection) আটকে রাখে।
- একটি Circuit Breaker-এর তিনটি state আছে — Closed (স্বাভাবিক), Open (দ্রুত ব্যর্থ, কোনো call পাঠানো হয় না), এবং Half-Open (সীমিত probe request) — যা একটি failure-rate threshold, একটি open-state timeout, এবং একটি half-open success threshold দ্বারা চালিত হয়।
- Retry গুলোকে synchronized thundering-herd retry এড়াতে jitter সহ exponential backoff ব্যবহার করতে হবে, এবং একটি retry budget দ্বারা সীমাবদ্ধ থাকা উচিত।
- Retry শুধুমাত্র idempotent অপারেশনের জন্য নিরাপদ; non-idempotent অপারেশন নিরাপদে retry করার জন্য idempotency key দরকার।
- Bulkhead প্রতিটি dependency-র জন্য resource pool (thread pool, connection pool, semaphore) আলাদা করে যাতে একটি ব্যর্থ dependency বাকি system-কে অনাহারে রাখতে না পারে, ঠিক জাহাজের watertight compartment-এর মতো।
- Production-এ, bulkhead, circuit breaker, এবং retry একত্রিত করা হয় — retry কখনোই একটি Open circuit-এর বিরুদ্ধে চালু হওয়া উচিত নয়।
- Resilience4j (Netflix Hystrix-এর উত্তরসূরি), AWS SDK-এর retry behavior, এবং Istio-এর circuit breaking এই প্যাটার্নগুলোর প্রকৃত, ব্যাপকভাবে ব্যবহৃত implementation।
