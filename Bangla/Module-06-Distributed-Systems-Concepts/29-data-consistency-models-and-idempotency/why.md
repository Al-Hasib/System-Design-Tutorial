# এই Topic-টি কেন গুরুত্বপূর্ণ: Data Consistency Models ও Idempotency

> **এক বাক্যে:** "Eventually consistent" একটি একক জিনিস নয় — এটি সুনির্দিষ্ট guarantee-এর একটি spectrum — এবং idempotency হলো একটিমাত্র technique যা প্রতিটি distributed system যে retry-এর উপর নির্ভর করে তা নিরাপদে সম্পাদন করা সম্ভব করে তোলে।

## এই ধারণার আগের পৃথিবী

**Consistency নিয়ে:** একটি team একটি design doc-এ "eventually consistent" লিখে এগিয়ে যায়। কেউ সংজ্ঞায়িত করে না *কতটা* eventually, বা এর মধ্যবর্তী সময়ে কী কী anomaly সম্ভব। তারপর user-রা এমন bug report করে যা অসম্ভব শোনায়: একটি comment যা post করার পর হারিয়ে যায়, একটি reply যা যে message-এর reply সেটার আগে দেখা যায়, একটি balance যা বেড়ে গিয়ে আবার কমে যায়। প্রতিটিকে bug হিসেবে investigate করা হয়। এদের কোনোটিই bug নয় — এগুলো হলো সুনির্দিষ্ট anomaly যা বেছে নেওয়া (unde­clared) consistency model অনুমতি দেয়।

**Idempotency নিয়ে:** একটি payment API timeout হয়। Client জানে না charge টি সম্পন্ন হয়েছে কিনা, তাই এটি retry করে। এটি সম্পন্ন হয়েছিল। Customer-কে দুইবার charge করা হয়। Team একটি check যোগ করে — "যদি সাম্প্রতিক charge থাকে তাহলে charge করবেন না" — যা racy এবং concurrency-এর অধীনে fail করে। Support manual-ভাবে refund handle করে। এটি production system-এ সবচেয়ে সাধারণ গুরুতর bug-গুলোর একটি, এবং এর একটি standard, সুপরিচিত সমাধান আছে যা team প্রয়োগ করতে জানত না।

## এটি যেসব সমস্যা সমাধান করে

### ১. User নিজের write অদৃশ্য হয়ে যেতে দেখে
**আপনি যা দেখেন:** একজন user তার profile edit করে, page reload হয়, এবং পুরনো value ফিরে আসে। আবার refresh করলে নতুন value দেখা যায়।

**কেন এমন হয়:** Write primary-তে গিয়েছিল; পরবর্তী read একটি lagging replica-তে গিয়ে পড়েছিল। সাধারণ eventual consistency ঠিক এটাই অনুমতি দেয়।

**Consistency model কীভাবে এটি সমাধান করে:** **Read-your-own-writes** একটি সুনির্দিষ্ট, নামকরণযোগ্য guarantee — একটি session সবসময় অন্তত তার নিজের write দেখে। একবার আপনি এটির নাম দিতে পারলে, আপনি এটি implement করতে পারেন: একটি write-এর পর একটি সংক্ষিপ্ত window-এর জন্য user-এর read-কে primary-তে route করুন, বা একটি version token pass করুন যা replica-কে catch up করতে হবে। Model-এর মূল্য theory-তে নয়; এটি একটি রহস্যকে সুপরিচিত implementation-সহ একটি requirement-এ পরিণত করে।

### ২. কারণের আগে প্রভাব দেখা দেওয়া
**আপনি যা দেখেন:** একটি comment thread-এ, একটি reply সেই comment-এর উপরে দেখা যায় যেটার সে reply। একটি chat-এ, প্রশ্নের আগেই একটি উত্তর আসে।

**কেন এমন হয়:** ভিন্ন ভিন্ন message ভিন্ন ভিন্ন path নেয় এবং out of order পৌঁছায়। Eventual consistency কোনো ordering প্রতিশ্রুতিই দেয় না।

**Causal consistency কীভাবে এটি সমাধান করে:** এটি guarantee করে যে যদি A causally B-এর আগে হয়, তাহলে সবাই B-এর আগে A দেখবে। Concurrent, unrelated event এখনও যেকোনো order-এ দেখা যেতে পারে — যা সস্তা — কিন্তু causality সংরক্ষিত থাকে, যা user-রা আসলে সঠিকতা হিসেবে উপলব্ধি করে। এটি social এবং messaging system-এর জন্য sweet spot, এবং এই কারণেই **logical clocks** (topic 41) এর অস্তিত্ব আছে।

### ৩. Retry যা বাস্তব-জগতের প্রভাব duplicate করে
**আপনি যা দেখেন:** Double charge, duplicate order, তিনবার পাঠানো email, একটি purchase-এর জন্য দুইবার inventory কমে যাওয়া।

**কেন এমন হয়:** একটি timeout মৌলিকভাবে অস্পষ্ট — client "request কখনো পৌঁছায়নি" এবং "এটি সফল হয়েছিল কিন্তু response হারিয়ে গিয়েছিল" এর মধ্যে পার্থক্য করতে পারে না। উভয়ই একই রকম দেখায়। এবং একটি distributed system-এ, আপনাকে *অবশ্যই* retry করতে হবে, কারণ timeout-এ হাল ছেড়ে দেওয়া মানে নিঃশব্দে বাস্তব কাজ হারিয়ে ফেলা।

**Idempotency কীভাবে এটি সমাধান করে:** Client request-এর সাথে একটি unique **idempotency key** পাঠায়। Server প্রথম execution-এর result-সহ সেই key record করে রাখে। সেই key-এর যেকোনো পুনরাবৃত্তি পুনরায় execute না করে stored result ফেরত দেয়। Operation-টি যতবার ইচ্ছা retry করার জন্য নিরাপদ হয়ে যায়। এই একটিমাত্র pattern production incident-এর একটি সম্পূর্ণ class দূর করে দেয়, এবং এই কারণেই Stripe, এবং মূলত প্রতিটি গুরুত্বপূর্ণ payments API, এটি require করে।

### ৪. Duplicate guarantee করা message queue
**আপনি যা দেখেন:** একটি consumer একটি rebalance বা acknowledgment-এর আগে crash-এর পর একই event দুইবার process করে।

**কেন এমন হয়:** কার্যত প্রতিটি queue at-least-once delivery প্রদান করে। Exactly-once হয় unavailable, না হয় ভারী constraint-সহ আসে।

**Idempotency কীভাবে এটি সমাধান করে:** At-least-once delivery-এর standard, সাদামাটা, সঠিক উত্তর হলো idempotent consumer — event ID দিয়ে deduplicate করা, বা operation-কে স্বভাবতই idempotent বানানো (একটি value increment করার বদলে set করা)। "আমরা exactly-once delivery ব্যবহার করব" সাধারণত এই ইঙ্গিত দেয় যে কেউ production-এ এটি এখনো সম্মুখীন হয়নি।

## যে মূল্য আপনাকে দিতে হয়

- **শক্তিশালী consistency-এর মূল্য latency ও availability।** Linearizability-এর জন্য প্রতিটি operation-এ coordination দরকার, যার মানে cross-node round trip এবং partition-এর সময় unavailability। প্রতিবারই আপনি গতির বিনিময়ে সঠিকতা কিনছেন।
- **দুর্বল consistency-এর মূল্য application complexity।** Anomaly-গুলো অদৃশ্য হয় না; সেগুলো আপনার code এবং আপনার UI-তে চলে যায়, যাকে এখন in-between state represent এবং tolerate করতে হয়।
- **Idempotency-এর জন্য storage এবং একটি retention policy দরকার।** Key-গুলো persist করতে হবে, প্রতিটি request-এ lookup করতে হবে (latency), এবং শেষপর্যন্ত expire করতে হবে — এবং খুব তাড়াতাড়ি expire করলে duplicate window আবার খুলে যায়।
- **Idempotency key সঠিকভাবে তৈরি করতে হবে।** Request content থেকে derive করা একটি key ভেঙে পড়ে যখন একজন user বৈধভাবে একই জিনিস দুইবার করতে চায়। প্রতিটি retry-তে নতুন করে তৈরি করা একটি key কিছুই করে না। Generation সঠিকভাবে করাটাই সেই অংশ যেখানে team ভুল করে।
- **একই key-সহ concurrent request একটি বাস্তব race।** ভিন্ন ভিন্ন server-এ একই সাথে আঘাত করা দুটি in-flight retry-এর জন্য শুধু check-then-act নয়, key-এর atomic reservation দরকার।

## কখন এটি দরকার — এবং কখন নয়

| Strong consistency যার জন্য | দুর্বল consistency যার জন্য যথেষ্ট |
|---|---|
| Account balance, checkout-এ inventory | View count, like count, follower count |
| Unique constraint (username, booking) | Feed, timeline, search index |
| Authorization এবং access control | Recommendation, analytics, trending |

| Idempotency যার জন্য বাধ্যতামূলক | কম গুরুত্বপূর্ণ যার জন্য |
|---|---|
| Payment, order, transfer — টাকা-সম্পর্কিত যেকোনো কিছু | শুধুমাত্র read |
| যেকোনো queue consumer (at-least-once স্বাভাবিক) | স্বভাবতই idempotent write (`SET status = 'active'`) |
| যেকোনো externally-triggered webhook handler | — |

## কেন এটি Interview-এ আসে

Idempotency system design interviewing-এর সবচেয়ে high-signal topic-গুলোর একটি। এটি না চাওয়া সত্ত্বেও উত্থাপন করা — "payment endpoint একটি idempotency key নেয়, কারণ client timeout-এ retry করবে এবং আমরা double-charge করতে পারি না" — সাথে সাথেই আপনাকে এমন কেউ হিসেবে চিহ্নিত করে যে বাস্তব system ship করেছে। Consistency দিক থেকে, interviewer-রা নির্ভুলতা চায়: শুধু "এটি eventually consistent" নয়, বরং *কোন* guarantee, কেন সেটি এই data-র জন্য যথেষ্ট, এবং user কী anomaly observe করতে পারে। এক নিঃশ্বাসে বলতে পারা "like eventually consistent, ledger linearizable, এবং comment thread-এর causal ordering দরকার" — এটাই ঠিক লক্ষ্য।

## এটি কীভাবে সংযুক্ত

Consistency model মোটা দাগের **CAP** (topic 15) এবং **ACID vs BASE** (topic 16) framing-কে এমন কিছুতে পরিমার্জিত করে যার বিরুদ্ধে আপনি আসলে design করতে পারেন। এগুলো **replication** lag (topic 13) এবং **caching** (topic 17)-এর সরাসরি ফলাফল। Idempotency হলো নিরাপদ **retry** (topic 26)-এর পূর্বশর্ত, **at-least-once** queue consumption (topic 20)-এর জন্য, এবং একটি **Saga** (topic 28)-এর প্রতিটি step ও compensation-এর জন্য। **Logical clocks** (topic 41) হলো সেই যন্ত্রপাতি যা causal consistency-কে implement-যোগ্য করে তোলে।

**পরবর্তী:** [Monolith vs Microservices](../../Module-07-Architecture-Patterns/30-monolith-vs-microservices/why.md) — সেই architectural সিদ্ধান্ত যা নির্ধারণ করে আপনাকে এর কতটা মোকাবিলা করতে হবে।
