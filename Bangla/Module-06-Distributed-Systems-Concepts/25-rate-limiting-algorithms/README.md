# Rate Limiting Algorithms (Token Bucket, Leaky Bucket, Sliding Window)

**কঠিনতা:** Advanced

## শেখার লক্ষ্যসমূহ (Learning Objectives)

- API এবং ব্যাকএন্ড রিসোর্স রক্ষার জন্য কেন rate limiting প্রয়োজন তা ব্যাখ্যা করা।
- Fixed Window, Sliding Window Log, Sliding Window Counter, Token Bucket, এবং Leaky Bucket অ্যালগরিদমগুলো তুলনা করা।
- অ্যালগরিদমগুলোর মধ্যে burst tolerance এবং traffic-smoothing এর ট্রেড-অফ নিয়ে সঠিকভাবে যুক্তি দেওয়া।
- Redis ব্যবহার করে অনেকগুলো API gateway বা service instance জুড়ে সামঞ্জস্যপূর্ণ থাকা একটি distributed rate limiter ডিজাইন করা।
- বাস্তব সিস্টেমগুলো (Stripe, AWS API Gateway, Nginx) কীভাবে প্রোডাকশনে rate limiting প্রয়োগ করে তা চিহ্নিত করা।

## স্ক্রিপ্ট (Script)

### শুরু / ভূমিকা (Hook / Intro)

কল্পনা করুন: আজ Black Friday, আপনার API-তে প্রচণ্ড চাপ পড়ছে, এবং একটি দুর্ব্যবহারকারী client script আপনার `/checkout` এন্ডপয়েন্টে সেকেন্ডে দশ হাজার request পাঠাচ্ছে। কোনো rate limiting নেই। আপনার database connection pool পরিপূর্ণ হয়ে যায়, প্রতিটি বৈধ কাস্টমারের জন্য latency বেড়ে যায়, এবং এখন একটি খারাপ actor-এর কারণে সবার দিন খারাপ যাচ্ছে। এটাই সেই সমস্যা যা rate limiting সমাধান করে। আজ আমরা গভীরভাবে দেখব সেই অ্যালগরিদমগুলো যা এটাকে কার্যকর করে — token bucket, leaky bucket, এবং sliding window পরিবার — এবং আমরা সেই বিষয়ে নিখুঁত হব যা বেশিরভাগ টিউটোরিয়াল হালকাভাবে এড়িয়ে যায়: লোড ব্যালেন্সারের পেছনে যখন একটি নয়, পঞ্চাশটি gateway instance থাকে, তখন কীভাবে ধারাবাহিকভাবে rate limit প্রয়োগ করবেন।

### কেন Rate Limiting গুরুত্বপূর্ণ (Why Rate Limiting Matters)

Rate limiting সীমাবদ্ধ করে যে একটি client — যাকে API key, user ID, বা IP দিয়ে চিহ্নিত করা হয় — একটি নির্দিষ্ট সময়ের window-এ কতগুলো request করতে পারবে। এই সিরিজে আপনি ইতিমধ্যে API gateway দেখেছেন; rate limiting সেখানে থাকা মূল দায়িত্বগুলোর একটি, auth এবং routing-এর পাশাপাশি। কেন এটা দরকার? তিনটি বড় কারণ। প্রথমত, fairness — একটি হৈচৈকারী tenant একই infrastructure ভাগ করে নেওয়া অন্য সব tenant-কে ক্ষুধার্ত রাখতে পারবে না। দ্বিতীয়ত, protection — এটা traffic spike, retry storm, এবং denial-of-service ধরনের অপব্যবহারের বিরুদ্ধে আপনার প্রথম প্রতিরক্ষা লাইন, ইচ্ছাকৃত হোক বা দুর্ঘটনাক্রমে। তৃতীয়ত, cost এবং capacity planning — যদি আপনি জানেন প্রতিটি client সর্বোচ্চ, ধরুন, প্রতি মিনিটে 100 request পাঠাতে পারবে, তাহলে আপনি আত্মবিশ্বাসের সাথে আপনার downstream service এবং database-এর আকার নির্ধারণ করতে পারবেন। আপনি যে অ্যালগরিদম বেছে নেন তা দুটি জিনিস নির্ধারণ করে: আপনি traffic-এর burst অনুমতি দেবেন কিনা, এবং আপনার ব্যাকএন্ডে পৌঁছানো traffic আসলে কতটা মসৃণ বা স্পাইকি দেখায়।

### Fixed Window Counter

সবচেয়ে সহজ পদ্ধতি: একটি window size বেছে নিন, ধরুন 60 সেকেন্ড, এবং প্রতিটি request-এ increment হওয়া প্রতি client-এর জন্য একটি counter রাখুন। যখন counter আপনার limit ছাড়িয়ে যায় — ধরুন 100 request — তখন window রিসেট না হওয়া পর্যন্ত আরও request প্রত্যাখ্যান করুন। একদম সহজ, প্রতি client-এ O(1) memory, একটিমাত্র Redis key এবং একটি TTL দিয়ে প্রয়োগ করা তুচ্ছ বিষয়। কিন্তু এর একটি সুপরিচিত ত্রুটি আছে: boundary problem। যদি একটি client এক window-এর শেষ সেকেন্ডে 100টি request পাঠায় এবং পরের window-এর প্রথম সেকেন্ডে আরও 100টি পাঠায়, তাহলে দুই সেকেন্ডের মধ্যে সেটা 200টি request, যদিও কনফিগার করা limit ছিল প্রতি মিনিটে 100। Fixed window কিছুই মসৃণ করে না — এটা শুধু প্রতি interval-এ একটি cliff রিসেট করে, এবং সেই boundary-তে ঠিক traffic প্রচণ্ডভাবে burst করতে পারে।

### Sliding Window Log এবং Sliding Window Counter

Sliding window log boundary problem সঠিকভাবে ঠিক করে: প্রতিটি client-এর জন্য, একটি sorted set-এ প্রতিটি request-এর একটি timestamp সংরক্ষণ করুন, এবং যখন একটি নতুন request আসে, তখন আপনার window-এর চেয়ে পুরনো entry বাদ দিন এবং যা বাকি থাকে তা গণনা করুন। এটা নিখুঁত — কোনো boundary artifact নেই — কিন্তু memory cost request volume অনুযায়ী বাড়ে, কারণ আপনি প্রতিটি request-এর জন্য একটি timestamp সংরক্ষণ করছেন। উচ্চ throughput-এ এটা ব্যয়বহুল। Sliding window counter হলো ব্যবহারিক সমঝোতা: প্রতিটি timestamp লগ করার পরিবর্তে, আপনি বর্তমান এবং পূর্ববর্তী fixed window-এর জন্য counter রাখেন, তারপর একটি weighted count হিসাব করেন — অনেকটা "current window count প্লাস previous window count গুণ previous window-এর যে অংশ এখনও sliding view-এর সাথে ওভারল্যাপ করছে তার fraction"। এটা fixed window counter-এর memory footprint দিয়ে log পদ্ধতির নির্ভুলতার কাছাকাছি পৌঁছায়। বেশিরভাগ প্রোডাকশন সিস্টেম আসলে এটাই ব্যবহার করে যখন তারা storage বৃদ্ধি ছাড়াই sliding-window semantics চায়।

### Token Bucket

এখন সেই অ্যালগরিদম যা বেশিরভাগ engineer ডিফল্ট হিসেবে বেছে নেয়: token bucket। একটি bucket কল্পনা করুন যাতে token থাকে, একটি capacity সহ — ধরুন 100 token। Token একটি স্থির হারে রিফিল হয়, ধরুন প্রতি সেকেন্ডে 10টি, সেই capacity পর্যন্ত। প্রতিটি আগত request-কে এগিয়ে যাওয়ার জন্য একটি token নিতে হয়; যদি bucket খালি থাকে, request প্রত্যাখ্যান বা queue করা হয়। এখানে মূল আচরণটি হলো: যদি client নিষ্ক্রিয় থাকে এবং bucket পূর্ণ থাকে, তাহলে এটা হঠাৎ 100টি request একসাথে burst-এ পাঠাতে পারে, তাৎক্ষণিকভাবে, এবং সেগুলো সব সফল হবে, কারণ token জমা হয়ে বসে ছিল। তারপর এটা প্রতি সেকেন্ডে 10-এর স্থির রিফিল হারে ফিরে যায়। Token bucket স্পষ্টভাবে bucket capacity পর্যন্ত burst-এর অনুমতি দেয় — অনেক বাস্তব-জগতের traffic pattern-এর জন্য এটা একটি বৈশিষ্ট্য, ত্রুটি নয়, যেমন একটি client যা মাঝেমধ্যে poll করে তারপর একদফা কাজ করে।

### Leaky Bucket

Leaky bucket মানসিক মডেলটি উল্টে দেয়। একটি প্রকৃত bucket কল্পনা করুন যার নিচে একটি ছোট ছিদ্র আছে, যেখান থেকে স্থির হারে পানি বের হচ্ছে। Request গুলো উপর থেকে ঢালা হয়, client যে হারে পাঠায় সেই হারে। যদি bucket পূর্ণ হয়ে যায় — অর্থাৎ request গুলো leak হারের চেয়ে দ্রুত আসে — তাহলে নতুন request গুলো উপচে পড়ে এবং বাদ দেওয়া হয় বা প্রত্যাখ্যান করা হয়। কিন্তু গুরুত্বপূর্ণভাবে, পানি একটি নির্দিষ্ট, স্থির হারে বের হয়, অর্থাৎ request গুলো প্রক্রিয়া করা হয়, ইনপুট যতই burst-প্রবণ হোক না কেন। এটা সাধারণত একটি FIFO queue হিসেবে প্রয়োগ করা হয় যা একটি fixed-rate worker দিয়ে খালি করা হয়। ফলাফল: leaky bucket traffic-কে একটি কঠোরভাবে সমরূপ outflow rate-এ মসৃণ করে। এটা leak rate-এর চেয়ে দ্রুত burst পাস করতে দেয় না — bucket-এ capacity থাকলেও, একটি হঠাৎ burst শুধু queue-তে জমা হয় এবং ধীরে ধীরে খালি হয়। এটাই token bucket থেকে মূল পার্থক্য: token bucket যতক্ষণ token জমা আছে ততক্ষণ burst তাৎক্ষণিকভাবে ফরওয়ার্ড করতে দেয়; leaky bucket সবসময় ইনপুট যেভাবেই আসুক না কেন একটি স্থির হারে আউটপুট দেয়। যখন বৈধ client থেকে burstiness সহ্য করতে চান তখন token bucket বেছে নিন; যখন আপনার downstream সিস্টেমের প্রকৃতপক্ষে একটি মসৃণ, স্থির-হার stream প্রয়োজন তখন leaky bucket বেছে নিন — যেমন একটি fixed-capacity queue-তে traffic shaping করা বা এমন একটি legacy সিস্টেম যা কোনো spike সামলাতে পারে না।

### Distributed Rate Limiting (একাধিক সার্ভার, Redis ব্যবহার করে)

এখানেই সেই অংশ যা আসলে একজন জুনিয়রের উত্তরকে একজন সিনিয়রের উত্তর থেকে আলাদা করে। এই সবকিছু সহজ যখন একটি মাত্র process memory-তে counter রাখে। কিন্তু আপনার API gateway একটি load balancer-এর পেছনে পঞ্চাশটি instance হিসেবে চলে, এবং একটি নির্দিষ্ট client-এর request গুলো যেকোনো instance-এ যেতে পারে। যদি প্রতিটি instance নিজের local token bucket রাখে, তাহলে একটি client gateway instance A দিয়ে 100টি request পার করতে পারে, তারপর instance B দিয়ে আরও 100টি, এবং আপনি নিঃশব্দে আপনার উদ্দিষ্ট limit-এর 5000% অনুমতি দিয়ে দিয়েছেন। আপনার শেয়ারড, সিঙ্ক্রোনাইজড state দরকার। প্রমিত সমাধান হলো Redis-এ একটি কেন্দ্রীভূত counter। কৌশলটি হলো atomicity: একটি counter increment করা এবং সেটাকে একটি limit-এর বিপরীতে চেক করা একটি read-then-write, এবং যদি আপনি সেটা দুটি আলাদা round trip হিসেবে করেন তাহলে concurrency-তে race condition তৈরি হয়। তাই আপনি fixed-window ধরনের limiting-এর জন্য Redis-এর atomic `INCR` একটি TTL সহ ব্যবহার করেন, অথবা, আরও দৃঢ়ভাবে, একটি Lua script যা token bucket বা sliding window counter logic সম্পূর্ণভাবে Redis-এর ভেতরে প্রয়োগ করে, `EVAL`-এর মাধ্যমে একটি single round trip-এ atomically execute হয়। Redis-এর single-threaded execution model নিশ্চিত করে কোনো interleaving হবে না। বিশেষভাবে token bucket-এর জন্য, Lua script শেষ রিফিলের পর কতটুকু সময় গেছে তা হিসাব করে, উপযুক্ত সংখ্যক token যোগ করে, capacity-তে সীমাবদ্ধ করে, এবং token পাওয়া গেলে atomically তা decrement করে — সবকিছু server-side। এটা আপনাকে সঠিক, বিশ্বব্যাপী সামঞ্জস্যপূর্ণ প্রয়োগ দেয়, কিন্তু এটা একটি network hop এবং প্রতিটি request-এর জন্য Redis উপলব্ধ ও low-latency থাকার উপর একটি নির্ভরতা যোগ করে। বিকল্প, যখন আপনি precision-এর বিনিময়ে কম latency এবং বেশি availability চান, তা হলো approximate local rate limiting: প্রতিটি gateway instance স্থানীয়ভাবে limit-ভাগ-instance-count প্রয়োগ করে, অথবা প্রতিটি request-এ চেক করার পরিবর্তে পর্যায়ক্রমে count একটি শেয়ারড store-এ sync করে। এটা eventually consistent — একটি client সংক্ষিপ্তভাবে global limit ছাড়িয়ে যেতে পারে — কিন্তু এটা প্রতিটি request-এর hot path থেকে Redis সরিয়ে দেয়। এই পছন্দটি একটি ক্লাসিক consistency-বনাম-latency ট্রেড-অফ, এবং আপনি কোনটি বেছে নেবেন তা নির্ভর করে আপনার use case-এর জন্য কয়েকশ মিলিসেকেন্ডের জন্য সামান্য অতিরিক্ত request অনুমোদন গ্রহণযোগ্য কিনা তার উপর।

### বাস্তব-জগতের উদাহরণ (Real-World Example)

এটা কল্পনা করার দরকার নেই — প্রোডাকশন সিস্টেমগুলো স্পষ্টভাবে এই ট্রেড-অফ করে। Stripe-এর API প্রতি account-এ একটি rate limit ডকুমেন্ট করে এবং `Retry-After` নির্দেশনা সহ একটি `429 Too Many Requests` ফেরত দেয়, কার্যকরভাবে একটি token-bucket-ধরনের মডেল যা সংক্ষিপ্ত burst সহ্য করার পাশাপাশি একটি স্থির দীর্ঘমেয়াদী হার প্রয়োগ করে। AWS API Gateway আপনাকে প্রতিটি API-তে একটি steady-state rate এবং একটি burst capacity উভয়ই কনফিগার করতে দেয়, যা আক্ষরিক অর্থে token bucket পরিভাষা সরাসরি console-এ উন্মুক্ত করা — rate হলো রিফিল rate, burst হলো bucket size। Nginx `ngx_http_limit_req_module`-এর মাধ্যমে rate limiting প্রয়োগ করে, যা ডিজাইনে একটি leaky bucket প্রয়োগ — এটা request গুলোকে queue এবং delay করে একটি কনফিগার করা হারে মসৃণ করার জন্য, একটি ঐচ্ছিক `burst` প্যারামিটার সহ যা একটি নিয়ন্ত্রিত সংখ্যক request-কে সরাসরি প্রত্যাখ্যাত না হয়ে queue করতে দেয়। লক্ষ্য করুন কীভাবে তিনটি বাস্তব সিস্টেমই token-bucket-এর মতো burst allowance-কে leaky-bucket-এর মতো smoothing-এর সাথে মিশিয়ে দেয় — বিশুদ্ধ পাঠ্যপুস্তক অ্যালগরিদমগুলো একটি সূচনাবিন্দু মাত্র, কিন্তু প্রোডাকশন কনফিগ সাধারণত উভয় থেকে ধারণা একত্র করে।

### সংক্ষিপ্তসার (Recap)

চলুন এটা একসাথে বেঁধে ফেলি। Fixed window সহজ কিন্তু এর একটি boundary burst সমস্যা আছে। Sliding window log নিখুঁত কিন্তু memory-ক্ষুধার্ত। Sliding window counter fixed-window-এর মতো memory cost দিয়ে log-কে আনুমানিক করে — অনেক টিমের জন্য ব্যবহারিক ডিফল্ট। Token bucket bucket capacity পর্যন্ত নিয়ন্ত্রিত burst-এর অনুমতি দেয়, তারপর একটি স্থির রিফিল হারে throttle করে — বৈধ burstiness সহ্য করার জন্য চমৎকার। Leaky bucket ইনপুট burstiness নির্বিশেষে একটি স্থির output rate বাধ্য করে — যখন downstream-এর সত্যিই মসৃণ, সমরূপ traffic দরকার তখন চমৎকার। এবং কঠিন distributed-systems অংশটি: এগুলোর যেকোনোটিকে অনেক stateless gateway instance জুড়ে সঠিকভাবে কাজ করানোর জন্য হয় Lua script সহ Redis-এর মতো একটি কেন্দ্রীভূত, atomically-আপডেট হওয়া store, অথবা কম latency-র জন্য local counter-এর মাধ্যমে একটি গৃহীত approximation প্রয়োজন।

### এরপর কী (What's Next)

Rate limiting আপনাকে অতিরিক্ত request থেকে রক্ষা করে। কিন্তু যখন একটি downstream service ধীর বা সম্পূর্ণভাবে ব্যর্থ হয়, এবং আপনার নিজের service তবুও তাকে কল করতে থাকে, পরিস্থিতি আরও খারাপ করে দেয়, তখন কী হবে? সেটাই পরবর্তী ভিডিও: Circuit Breaker, Retry, এবং Bulkhead pattern — কীভাবে service-to-service কলে resilience তৈরি করবেন যাতে একটি ব্যর্থ dependency সম্পূর্ণ outage-এ পরিণত না হয়। সেখানে দেখা হচ্ছে।

## মূল বিষয়সমূহ (Key Takeaways)

- Rate limiting প্রতি client-এর request rate সীমাবদ্ধ করে fairness, availability, এবং cost predictability রক্ষা করে।
- Fixed window সবচেয়ে সহজ কিন্তু উদ্দিষ্ট limit-এর সর্বোচ্চ 2 গুণ পর্যন্ত boundary burst-এর অনুমতি দেয়।
- Sliding window log নিখুঁত কিন্তু প্রতি client-এ memory-তে O(n); sliding window counter সস্তায় এটার আনুমানিক করে।
- Token bucket bucket capacity পর্যন্ত burst-এর অনুমতি দেয়, তারপর রিফিল হারে throttle করে — বৈধ burst-প্রবণ traffic-এর জন্য ভালো।
- Leaky bucket ইনপুট burstiness নির্বিশেষে একটি স্থির output rate প্রয়োগ করে — ভঙ্গুর downstream সিস্টেমে traffic মসৃণ করার জন্য ভালো।
- Distributed rate limiting-এর জন্য শেয়ারড, atomically-আপডেট হওয়া state প্রয়োজন (যেমন, Redis `INCR` বা Lua script) যাতে প্রতি-instance অতিরিক্ত অনুমোদন এড়ানো যায়; local approximation latency এবং availability-র বিনিময়ে consistency ছাড় দেয়।
- বাস্তব সিস্টেমগুলো (Stripe, AWS API Gateway, Nginx) একটি বিশুদ্ধ পাঠ্যপুস্তক অ্যালগরিদম ব্যবহার না করে burst allowance এবং smoothing মিশিয়ে দেয়।
