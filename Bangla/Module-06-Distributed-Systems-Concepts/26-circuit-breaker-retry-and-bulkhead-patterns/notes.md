# স্টাডি নোটস — Circuit Breaker, Retry & Bulkhead Patterns

## সংজ্ঞাসমূহ

- **Circuit Breaker**: একটি remote call-এর চারপাশে একটি stateful proxy যা failure rate নিরীক্ষণ করে এবং একবার একটি threshold অতিক্রম করলে, dependency পুনরুদ্ধারের সময় না পাওয়া পর্যন্ত একটি সমস্যাগ্রস্ত dependency-তে request পাঠানো বন্ধ করে দেয় ("fail fast")।
- **Retry**: একটি ব্যর্থ অপারেশন স্বয়ংক্রিয়ভাবে পুনরায় চেষ্টা করা, সাধারণত একটি delay কৌশল সহ, যাতে caller-এর কাছে না জানিয়ে ক্ষণস্থায়ী/স্বল্পস্থায়ী failure গুলো সামলানো যায়।
- **Bulkhead**: প্রতিটি dependency বা workload-এর জন্য resource (thread pool, connection pool, semaphore) আলাদা করা যাতে একটি partition-এর exhaustion বাকিগুলোকে অনাহারে রাখতে না পারে — জাহাজের watertight compartment থেকে নামকরণ করা হয়েছে।
- **Cascading failure**: একটি component-এর failure (যেমন, একটি ধীরগতির dependency) যা calling component গুলোতে shared resource (thread, connection, queue) শেষ করে উপরের দিকে ছড়িয়ে পড়ে।
- **Idempotency**: একটি অপারেশনের একটি বৈশিষ্ট্য যেখানে এটি একাধিকবার প্রয়োগ করলে একবার প্রয়োগ করার মতোই একই প্রভাব পড়ে — non-read অপারেশনের নিরাপদ retry-এর জন্য একটি পূর্বশর্ত।
- **Thundering herd**: অনেক client একই synchronized মুহূর্তে ব্যর্থ হয় এবং retry করে, ইতিমধ্যে সমস্যাগ্রস্ত system-এর উপর load বাড়িয়ে দেয়।

## Circuit Breaker State Table

| State | প্রবেশের ট্রিগার | State-এ থাকা অবস্থায় আচরণ |
|---|---|---|
| Closed | প্রাথমিক state / Half-Open সফলতার পরে breaker reset | Request গুলো স্বাভাবিকভাবে চলে; failure গুলো একটি rolling window জুড়ে (count বা time-based) গণনা করা হয় |
| Open | Closed থাকা অবস্থায় failure rate (বা পরপর failure) নির্ধারিত threshold অতিক্রম করে | সব call dependency-র সাথে যোগাযোগ না করেই সাথে সাথে ব্যর্থ হয়; একটি timeout ("sleep window") timer শুরু হয় |
| Half-Open | Open-state timeout শেষ হয় | পুনরুদ্ধার যাচাই করতে সীমিত সংখ্যক probe/test request যেতে দেওয়া হয় |
| → Closed | Half-Open-এ probe request গুলো success threshold পূরণ করে | Breaker failure counter reset করে, স্বাভাবিক traffic পুনরায় শুরু করে |
| → Open | Half-Open-এ যেকোনো (বা যথেষ্ট) probe request ব্যর্থ হয় | Breaker আবার open হয় এবং timeout timer পুনরায় শুরু করে |

মূল tuning parameter: failure-rate threshold, rolling window-এর আকার, open-state timeout-এর সময়কাল, half-open probe count, half-open success threshold।

## Retry কৌশলের তুলনা

| কৌশল | Delay প্যাটার্ন | সুবিধা | অসুবিধা |
|---|---|---|---|
| Fixed delay | প্রতিটি চেষ্টার মধ্যে একই অপেক্ষার সময় (যেমন, সবসময় ৫০০ms) | বাস্তবায়ন এবং বোঝা সহজ | ধারাবাহিক failure-এর সময় back off করে না; thundering herd-এ অবদান রাখতে পারে |
| Exponential backoff | প্রতিটি চেষ্টায় delay দ্বিগুণ হয় (বা exponentially বাড়ে) (১০০ms, ২০০ms, ৪০০ms, ৮০০ms…) একটি সর্বোচ্চ সীমা পর্যন্ত | একটি পুনরুদ্ধারকারী/overloaded dependency-কে ক্রমবর্ধমানভাবে বেশি শ্বাস নেওয়ার জায়গা দেয় | যদি অনেক client একই সাথে ব্যর্থ হয়, তারা synchronized wave-এ retry করে |
| Exponential backoff + jitter | Exponential window-এর মধ্যে (বা তার আশেপাশে) random delay (যেমন, AWS "full jitter": random(0, backoff)) | সময় জুড়ে retry ছড়িয়ে দেয়, synchronized thundering herd এড়ায় | প্রতি client-এর সময় সামান্য কম predictable, কিন্তু system-ব্যাপী load-এর জন্য নিঃসন্দেহে ভালো |

অন্যান্য অপরিহার্য retry বিষয়:
- মোট চেষ্টার সংখ্যা সীমাবদ্ধ করুন এবং/অথবা একটি **retry budget** ব্যবহার করুন (যেমন, retry ≤ মোট request volume-এর X%)।
- Retry শুধুমাত্র **idempotent** অপারেশনে নিরাপদ, অথবা server-এ একটি **idempotency key** দ্বারা সুরক্ষিত non-idempotent অপারেশনে।
- একটি **Open** circuit breaker-এর বিরুদ্ধে কখনো retry করবেন না — breaker-এর fail-fast আচরণ বজায় থাকতে দিন।
- Retryable error (timeout, 5xx, throttling) কে non-retryable error (4xx validation error, auth failure) থেকে আলাদা করুন।

## Bulkhead Isolation কৌশলসমূহ

- **Thread pool isolation** — প্রতিটি downstream dependency তার নিজস্ব সীমিত executor/thread pool পায় যাতে এর exhaustion অন্য call-এর জন্য প্রয়োজনীয় thread গুলোকে অনাহারে রাখতে না পারে।
- **Connection pool isolation** — প্রতিটি downstream target-এর জন্য আলাদা, সীমিত DB/HTTP connection pool।
- **Semaphore isolation** — একটি হালকা concurrency counter যা একটি dedicated thread pool-এর overhead ছাড়াই একটি dependency-তে in-flight call সীমিত করে (খুব উচ্চ throughput-এ উপযোগী)।
- **আলাদা service instance / node pool** — infrastructure স্তরে workload class বা tenant আলাদা করা যাতে একটি noisy consumer পুরো shared capacity গ্রাস করতে না পারে।
- **Queue isolation** — একটি shared unbounded queue-এর বদলে প্রতিটি dependency-র জন্য dedicated, bounded request queue।

## Interview পুনর্বিবেচনা — দ্রুত পয়েন্টসমূহ

- Circuit breaker = fail fast + স্বয়ংক্রিয়-পুনরুদ্ধার probing; তিনটি state: Closed, Open, Half-Open।
- Retry = ক্ষণস্থায়ী failure সামলানো; অবশ্যই exponential backoff + jitter + retry budget + idempotency ব্যবহার করতে হবে।
- Bulkhead = প্রতিটি dependency-র জন্য resource isolation; একটি failure domain-কে অন্যদের অনাহারে রাখতে বাধা দেয়।
- তিনটিই একত্রিত হয়: bulkhead concurrency সীমিত করে → circuit breaker একটি পরিচিত-খারাপ dependency-তে অপচয়িত call এড়ায় → retry (backoff/jitter সহ) অবশিষ্ট ক্ষণস্থায়ী সমস্যা সামলায়, কিন্তু কখনো একটি Open breaker-এর বিরুদ্ধে retry করে না।
- প্রকৃত implementation: resilience4j (Java, Hystrix-এর উত্তরসূরি), AWS SDK-এর built-in retry/backoff, service mesh স্তরে Istio-এর outlier detection/circuit breaking।
