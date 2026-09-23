# এই বিষয়টি কেন গুরুত্বপূর্ণ: Circuit Breaker, Retry & Bulkhead Patterns

> **এক বাক্যে:** একটি distributed system-এ, যা আপনাকে ফেলে দেয় তা কদাচিৎ মূল failure-টি নিজেই — এটি আপনার নিজের code সেই failure-এ যেভাবে সাড়া দেয় তার কারণে, একটি সমস্যাগ্রস্ত dependency-কে retry দিয়ে আঘাত করতে থাকা যতক্ষণ না পুরো system ভেঙে পড়ে।

## এই ধারণার আগের পৃথিবী

একটি recommendation service ধীরগতির হয়ে যায় — down নয়, শুধু ধীর, ৫০ ms-এর বদলে ৩০ সেকেন্ড সময় নিচ্ছে। কোনো protection ছাড়া একটি system-এ এরপর যা ঘটে তা এখানে দেখুন:

আপনার product page এটিকে synchronously কল করে। প্রতিটি request এখন ৩০ সেকেন্ডের জন্য একটি thread ধরে রাখছে। আপনার server-এ ২০০টি thread আছে। প্রতি সেকেন্ডে ১০০টি request-এ, দুই সেকেন্ডের মধ্যেই সব ২০০টি thread খরচ হয়ে যায়, এবং তারা সবাই এমন একটি service-এর জন্য অপেক্ষা করছে যেটি সাড়া দিতে যাচ্ছে না। নতুন request গুলো queue-তে জমা হয়, তারপর timeout হয়। আপনার product page এখন down — সম্পূর্ণভাবে — কারণ একটি *পরিপূরক* feature ধীরগতির হয়ে গেছে।

এদিকে আপনার client library timeout-এ তিনবার retry করে। তাই একটি প্রতি সেকেন্ডে ১,০০০ request সামলানো সমস্যাগ্রস্ত service এখন ৪,০০০ পাচ্ছে। এর পুনরুদ্ধারের একটি সুযোগ ছিল; এখন কোনো সুযোগ নেই। আপনার load balancer আপনার server গুলোকে unhealthy চিহ্নিত করে এবং rotation থেকে বাদ দিয়ে দেয়, বাকিগুলোর উপর traffic কেন্দ্রীভূত করে, যা একইভাবে ব্যর্থ হয়। চার মিনিটের মধ্যে, সবকিছু down।

সেই ক্রমের কোনোকিছুই একটি bug নয়। এটি স্বাভাবিক দেখতে code-এর প্রাকৃতিক, ডিফল্ট আচরণ।

## এটি যে সমস্যাগুলো সমাধান করে

### ১. একটি ধীরগতির dependency থেকে thread এবং connection exhaustion
**আপনি যা দেখবেন:** আপনার service সাড়া দিচ্ছে না; profiling দেখায় প্রতিটি worker একটি downstream call-এ আটকে আছে।

**কেন এটি ঘটে:** Unbounded অপেক্ষা। কোনো timeout ছাড়া, বা একটি উদার timeout সহ একটি call, একটি downstream latency সমস্যাকে একটি upstream availability সমস্যায় রূপান্তরিত করে।

**timeout এবং bulkhead কীভাবে এটি সমাধান করে:** আক্রমণাত্মক timeout সীমা দেয় কতক্ষণ কোনো request একটি resource ধরে রাখতে পারে। **Bulkhead** আরও এগিয়ে যায়: প্রতিটি dependency-কে তার নিজস্ব সীমিত connection pool বা thread pool দিন, যাতে recommendation service যাই হোক না কেন সর্বোচ্চ ২০টি thread খরচ করতে পারে। নামটি এসেছে জাহাজের compartment থেকে — একটি প্লাবিত অংশ পুরো জাহাজকে ডুবিয়ে দেয় না।

### ২. Retry যা নিশ্চিত করে dependency কখনো পুনরুদ্ধার হবে না
**আপনি যা দেখবেন:** একটি সংক্ষিপ্ত blip একটি দীর্ঘ outage-এ পরিণত হয়। ব্যর্থ service-এর দিকে traffic স্বাভাবিকের চেয়ে বহুগুণ বেশি।

**কেন এটি ঘটে:** প্রতিটি client একই সাথে retry করে। Retry storm একটি positive feedback loop: failure retry ঘটায়, retry আরও বেশি failure ঘটায়।

**smart retry কীভাবে এটি সমাধান করে:** Exponential backoff সময় জুড়ে চেষ্টাগুলো ছড়িয়ে দেয়; **jitter** (delay-কে randomize করা) হলো সেই গুরুত্বপূর্ণ সংযোজন যা সব client-কে synchronized wave-এ retry করা থেকে বিরত রাখে। Retry budget মোট traffic-এর একটি ভগ্নাংশ হিসেবে total retry সীমাবদ্ধ করে। এবং সবচেয়ে গুরুত্বপূর্ণ: শুধুমাত্র idempotent অপারেশন retry করুন, নাহলে আপনি একটি charge-কে তিনটিতে পরিণত করবেন।

### ৩. এমন কিছুতে অর্থহীন call যা আপনি জানেন ভাঙা আছে
**আপনি যা দেখবেন:** প্রতি সেকেন্ডে হাজার হাজার request, প্রতিটি একটি সম্পূর্ণ timeout-এর জন্য অপেক্ষা করছে, এমন একটি service-এর বিরুদ্ধে যা পাঁচ মিনিট ধরে শুধু error ছাড়া কিছু return করেনি।

**কেন এটি ঘটে:** প্রতিটি request স্বাধীনভাবে failure-টি পুনরায় আবিষ্কার করে, প্রতিবার পূর্ণ latency খরচ দিয়ে।

**circuit breaker কীভাবে এটি সমাধান করে:** একটি failure threshold-এর পরে breaker *open* হয় এবং call গুলো network স্পর্শ না করেই সাথে সাথে ব্যর্থ হয়। দুটি সুবিধা, উভয়ই বড়: আপনার service অপেক্ষা করে resource অপচয় করা বন্ধ করে, এবং সমস্যাগ্রস্ত dependency পুনরুদ্ধারের জন্য একটি প্রকৃত বিরতি পায়। একটি cooldown-এর পরে breaker *half-open* হয়ে যায় এবং পুনরুদ্ধার পরীক্ষা করতে সামান্য traffic যেতে দেয়, সফল হলে সম্পূর্ণভাবে বন্ধ হয়ে যায়। এটি একটি automated, দ্রুত circuit — electrical circuit-এর মতোই একই ধারণা।

### ৪. অপ্রয়োজনীয় feature অত্যাবশ্যক feature-কে ফেলে দিচ্ছে
**আপনি যা দেখবেন:** loyalty-points service down থাকার কারণে checkout ব্যর্থ হয়।

**কেন এটি ঘটে:** প্রতিটি dependency-কে required হিসেবে বিবেচনা করা হয় কারণ কোনোকিছুই critical এবং optional-এর মধ্যে পার্থক্য করে না।

**graceful degradation কীভাবে এটি সমাধান করে:** একটি breaker-এর সাথে মিলিয়ে, একটি fallback আপনাকে recommendation ছাড়াই page serve করতে, checkout সম্পূর্ণ করতে এবং পরে points প্রদান করতে, বা live data-এর বদলে cached data দেখাতে দেয়। dependency গুলোকে critical বনাম optional হিসেবে শ্রেণীবদ্ধ করা — এবং optional path-টি code করা — এটাই একটি আংশিক failure-কে আংশিক রাখে।

## যে মূল্য আপনাকে দিতে হবে

- **Tuning প্রকৃতপক্ষে কঠিন এবং কখনো শেষ হয় না।** Timeout খুব ছোট হলে আপনি এমন request ব্যর্থ করবেন যা সফল হতে পারত; খুব দীর্ঘ হলে আপনি কিছুই রক্ষা করছেন না। Breaker threshold অতিরিক্ত sensitive হলে এটি স্বাভাবিক variance-এ trip করে; খুব শিথিল হলে এটি কখনো সাহায্য করে না। এই value গুলোর জন্য প্রকৃত latency data এবং পর্যায়ক্রমিক পুনর্বিবেচনা প্রয়োজন।
- **Breaker outage ঘটাতে পারে।** একটি ভুল configure করা breaker যা একটি ক্ষণস্থায়ী blip-এ open হয় তা একটি সুস্থ dependency-কে অপ্রাপ্য করে তোলে। এটি একটি বাস্তব, পুনরাবৃত্ত production incident category।
- **Retry-এর idempotency দরকার, যা বিনামূল্যে নয়।** অপারেশনগুলোকে পুনরাবৃত্তির জন্য নিরাপদ করতে idempotency key এবং deduplication storage প্রয়োজন।
- **Fallback হলো এমন code যা শুধুমাত্র incident-এর সময় চলে** — যার মানে এটি আপনার সবচেয়ে কম-পরীক্ষিত code, এবং এটি সবচেয়ে খারাপ মুহূর্তে ব্যর্থ হয় যদি না আপনি ইচ্ছাকৃতভাবে এটি অনুশীলন করেন (এটি chaos engineering-এর অন্যতম সেরা যুক্তি)।
- **সর্বত্র জটিলতা।** এখন প্রতিটি call site-এ timeout, retry, breaker, এবং fallback policy আছে। Service mesh (Envoy, Istio) মূলত এটি application code থেকে সরানোর জন্য বিদ্যমান, একটি mesh চালানোর খরচে।

## কখন এটি প্রয়োজন — এবং কখন প্রয়োজন নেই

| এগুলো প্রয়োগ করুন যখন | আপনি সহজ রাখতে পারেন যখন |
|---|---|
| আপনি নিয়ন্ত্রণ করেন না এমন একটি service-এ যেকোনো network call | In-process function call |
| একটি dependency optional বা degradable | একটি local database সহ একটি একক monolith |
| একটি feature-এর failure cascade হতে হবে না | Failure বিরল এবং blast radius একজন user |
| আপনার একে অপরকে কল করা অনেক service আছে | — |

**তবে সবসময় timeout সেট করুন।** কোনো timeout ছাড়া একটি call system-এর আকার নির্বিশেষে একটি সুপ্ত outage।

## Interview-এ এটি কেন আসে

"এই service down হলে কী হয়?" প্রায় প্রতিটি design round-এ জিজ্ঞাসা করা হয়, এবং প্রত্যাশিত উত্তর "এটি retry করে" নয়। শক্তিশালী উত্তরগুলো পুরো chain কভার করে: timeout, exponential backoff *এবং jitter* সহ bounded retry, রক্তক্ষরণ বন্ধ করতে circuit breaker, resource consumption সীমাবদ্ধ করতে bulkhead, এবং একটি fallback বা degraded experience যাতে user তবুও কিছু পায়। বিশেষভাবে jitter উল্লেখ করা, এবং retry-এর জন্য idempotency প্রয়োজন তা লক্ষ্য করা, উভয়ই production অভিজ্ঞতার শক্তিশালী সংকেত।

## এটি কীভাবে সংযুক্ত

এই প্যাটার্নগুলো টপিক ৫-এর **fault tolerance** ধারণাগুলোর ব্যবহারিক বাস্তবায়ন। retry নিরাপদ করতে এগুলো **idempotency**-এর (টপিক ২৯) উপর নির্ভর করে। এগুলো **rate limiting**-কে (টপিক ২৫) পরিপূরক করে — সেটি আপনাকে caller থেকে রক্ষা করে, এগুলো আপনাকে callee থেকে রক্ষা করে। এগুলো **microservices communication**-এ (টপিক ৩১) অপরিহার্য, সাধারণত **API gateway**-এ (টপিক ৯) বা একটি service mesh-এ প্রয়োগ করা হয়, **tracing**-এর (টপিক ৪৩) মাধ্যমে পর্যবেক্ষণযোগ্য করা হয়, এবং **chaos engineering** (টপিক ৪৬) দ্বারা যাচাই করা হয়।

**পরবর্তী:** [Consensus Algorithms: Paxos & Raft](../27-consensus-algorithms-paxos-and-raft/why.md) — কীভাবে অবিশ্বস্ত machine-এর একটি দল আদৌ কোনোকিছুতে সম্মত হয়।
