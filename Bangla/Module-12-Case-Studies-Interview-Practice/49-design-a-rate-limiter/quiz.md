# Follow-Up Interview Questions — Design a Rate Limiter

[`README.md`](./README.md)-এর মূল script-এর বাইরেও পুনরাবৃত্তি করতে এগুলো ব্যবহার করুন। প্রতিটি উত্তর ইচ্ছাকৃতভাবে সংক্ষিপ্ত — একটি বাস্তব interview-এ জোরে জোরে বিস্তারিত বলুন।

---

**১. একটি অঞ্চলের একাধিক gateway instance-এর পরিবর্তে একাধিক data center জুড়ে আপনি কীভাবে rate-limit করবেন?**

region জুড়ে একটি global, strongly-consistent counter রাখার চেষ্টা করবেন না — cross-region round trip আপনার latency budget উড়িয়ে দেবে। পরিবর্তে, প্রতিটি data center-কে global limit-এর একটি per-region share সহ একটি local Redis cluster দিন (যেমন, global limit ÷ region-এর সংখ্যা, প্রত্যাশিত regional traffic দিয়ে weighted), এবং ঐচ্ছিকভাবে ব্যাকগ্রাউন্ডে asynchronously reconcile করুন। এটি perfect global precision-কে regional low latency-র জন্য বিনিময় করে — একই availability-over-consistency সিদ্ধান্ত যা single-region design-এ ছিল, শুধু এক level উপরে প্রয়োগ করা হয়েছে।

---

**২. Redis বন্ধ হয়ে গেলে কী হয়? পুরো failure-টি শুরু থেকে শেষ পর্যন্ত ব্যাখ্যা করুন।**

Gateway থেকে Redis-এ call গুলো একটি circuit breaker-এ wrap করা থাকে। যথেষ্ট সংখ্যক পরপর failure/timeout-এর পর, breaker trip করে এবং Redis কল করা বন্ধ করে দেয়, latency জমা হওয়ার বদলে দ্রুত ব্যর্থ হয়। তারপর policy কার্যকর হয়: বেশিরভাগ public/read traffic-এর জন্য fail-open (একটি conservative local in-memory fallback limit ব্যবহার করে request গুলোকে যেতে দেওয়া) যেখানে availability বেশি গুরুত্বপূর্ণ; login-এর মতো sensitive endpoint-এর জন্য fail-closed (request reject করা), যেখানে একটি outage-এর সময় unlimited traffic যেতে দেওয়া বিপজ্জনক। breaker পর্যায়ক্রমে Redis-কে probe করে (half-open state) recovery শনাক্ত করতে এবং normal enforcement পুনরায় শুরু করতে।

---

**৩. Sliding window বনাম fixed window — fixed window-এ boundary-তে নির্দিষ্টভাবে কী ভুল হয়?**

Fixed window একটি নির্দিষ্ট interval boundary-তে counter-কে শূন্যে reset করে। একজন client boundary-র ঠিক আগে পুরো limit পাঠাতে পারে এবং boundary-র ঠিক পরে আবার পুরো limit পাঠাতে পারে, reset-কে ঘিরে একটি সংক্ষিপ্ত span-এর মধ্যে নির্ধারিত limit-এর 2x পর্যন্ত উৎপন্ন করে — কারণ দুটি burst দুটি ভিন্ন "window"-এ পড়ে যা counter-এর দৃষ্টিতে overlap করে না, যদিও বাস্তব সময়ে তারা overlap করে।

---

**৪. Sliding window counter প্রতিটি timestamp সংরক্ষণ না করে sliding window log-কে কীভাবে approximate করে?**

এটি দুটি fixed-window counter রাখে — current এবং previous — এবং একটি weighted estimate গণনা করে: current window count যোগ previous window count গুণিত previous window-এর সেই অংশ যা এখনো sliding view-এর ভেতরে আছে। এটি একটি approximation, exact নয়, কিন্তু এটি প্রকৃত sliding-window আচরণের খুব কাছাকাছি পৌঁছায় যখন এটি প্রতি request O(n)-এর পরিবর্তে প্রতি key memory-তে O(1) থাকে।

---

**৫. আপনার কি একটি global rate limit, একটি per-endpoint limit, নাকি উভয়ই কার্যকর করা উচিত?**

সাধারণত উভয়ই। একটি global per-client limit সামগ্রিক fairness এবং cost রক্ষা করে, যখন per-endpoint override নির্দিষ্ট hot বা sensitive path (যেমন, `/login`, `/search`, একটি ভারী report-generation endpoint) রক্ষা করে যাদের account-এর baseline-এর চেয়ে কঠোর বা শিথিল limit দরকার। per-endpoint-কে global check-এর উপরে একটি layered override হিসেবে implement করুন, এর প্রতিস্থাপন হিসেবে নয়।

---

**৬. ভালো ব্যবহারকারীদের reject না করে বা abuse-কে যেতে না দিয়ে আপনি কীভাবে বৈধ traffic burst handle করবেন?**

যখন burst tolerance একটি requirement, fixed/sliding window-এর বদলে token bucket ব্যবহার করুন — এটি স্পষ্টভাবে একজন client-কে এক burst-এ bucket-এর পুরো capacity খরচ করার অনুমতি দেয় যদি সে idle থেকে token জমিয়ে থাকে, তারপর steady refill rate-এ throttle করে। এটিকে sustained rate-এর চেয়ে সামান্য উচ্চতর একটি short-term burst allowance-এর সাথে জোড়া দিন (একই rate + burst split যা AWS API Gateway প্রকাশ করে) যাতে একটি বৈধ মুহূর্তিক spike-কে sustained abuse-এর মতো শাস্তি দেওয়া না হয়।

---

**৭. আপনি malicious traffic-কে একটি বৈধ spike (যেমন, একটি flash sale) থেকে কীভাবে আলাদা করবেন?**

শুধু request-rate limiting intent সম্পূর্ণভাবে আলাদা করতে পারে না — এর জন্য rate-এর বাইরের সংকেত দরকার: request pattern (uniform bot-like timing বনাম human variability), source diversity (একটি IP/API key বনাম অনেক ভিন্ন বৈধ ব্যবহারকারী), এবং rate limiting-এর উপরে layered behavioral anomaly detection। একটি interview-এ, এটা বলা যুক্তিসঙ্গত যে rate limiter-এর কাজ হলো intent নির্বিশেষে ক্ষতি সীমিত করা, যখন একটি আলাদা abuse-detection/WAF layer classification সমস্যাটি সামলায় — এবং প্রত্যাশিত বৈধ spike (একটি flash sale) limiter-কে "অনুমান" করতে দেওয়ার বদলে একটি pre-provisioned উচ্চতর limit tier পাওয়া উচিত।

---

**৮. Application থেকে আলাদা GET/INCR call-এর বদলে Redis-এ একটি atomic Lua script কেন ব্যবহার করবেন?**

একটি আলাদা read-then-write দুটি network round trip যার মাঝখানে একটি race window আছে — concurrency-র অধীনে, দুটি request উভয়েই limit-এর ঠিক নিচে একটি count পড়তে পারে এবং উভয়ই admit হতে পারে, নীরবে limit ছাড়িয়ে যায়। `EVAL`-এর মাধ্যমে execute হওয়া একটি Lua script সম্পূর্ণভাবে Redis-এর single-threaded execution engine-এর ভেতরে একটি atomic unit হিসেবে চলে, তাই read, refill/window math, এবং decrement কখনো অন্য client-এর script-এর সাথে interleave হতে পারে না।

---

**৯. একটি timed-out request retry করার সময় একজন client-এর quota দ্বিগুণ charge করা আপনি কীভাবে এড়াবেন?**

Retry-যোগ্য request গুলোকে একটি idempotency key দিয়ে tag করুন। Rate limiter (বা এর পেছনের service) একটি bounded window-এর মধ্যে একটি পুনরাবৃত্ত key-কে একটি নতুন request হিসেবে না গণনা করে ইতিমধ্যে-গণনা-করা একটি request-এর duplicate হিসেবে চেনে, তাই একটি network-level retry logically একটি single operation-এর জন্য দুই একক quota consume করে না। এটি একই idempotency mechanism যা সাধারণভাবে নিরাপদ retry-র জন্য ব্যবহৃত হয়, শুধু নির্দিষ্টভাবে quota-counting path-এ প্রয়োগ করা হয়েছে।

---

**১০. এখানে hot-key সমস্যাটি কী, এবং আপনি এটি কীভাবে প্রশমিত করবেন?**

একটি ভালোভাবে-sharded Redis Cluster থাকলেও, একটি নির্দিষ্ট key-র (একজন viral account, একটি ভারী-ব্যবহৃত API key) সব request একটি shard-এ hash হয়, তাই মোট shard যতই থাকুক না কেন সেই shard-এর load ছড়িয়ে পড়ে না। প্রতিকার: প্রতি request-এ একটি Redis round trip-এর বদলে Redis-এ periodic sync সহ প্রতিটি gateway instance-এ local pre-aggregation, একটি hot logical key-কে একটি ছোট in-process reconciliation সহ কয়েকটি sub-key-তে ভাগ করা, অথবা পরিচিত উচ্চ-volume key-দের একটি dedicated, over-provisioned shard দেওয়া।

---

**১১. Gateway instance জুড়ে clock skew একটি distributed rate limiter-কে কীভাবে প্রভাবিত করে, এবং আপনি এটি কীভাবে এড়াবেন?**

যদি প্রতিটি gateway instance তার নিজের local clock ব্যবহার করে elapsed time বা window boundary গণনা করে, instance-গুলোর মধ্যে skew অসামঞ্জস্যপূর্ণ refill/reset timing ঘটাতে পারে — একটি instance মনে করে একটি window অন্যটির সামান্য আগে roll over হয়েছে। সমাধান হলো সব time-dependent logic (শেষ refill থেকে elapsed time, current window index) atomic Lua script-এর ভেতরে Redis-এর নিজস্ব server clock ব্যবহার করে গণনা করা, যাতে NTP-synced-কিন্তু-তবুও-অপূর্ণ client clock বিশ্বাস করার বদলে সময়ের একটি একক source of truth থাকে।

---

**১২. আপনি কোথায় rate limit কার্যকর করবেন: client SDK, API gateway, নাকি service mesh — এবং কেন শুধু একটি বেছে নেবেন না?**

প্রতিটি layer একটি ভিন্ন সমস্যা সমাধান করে। একটি client SDK একটি local, optimistic limit কার্যকর করে তাৎক্ষণিক feedback দিতে এবং একটি request-এ network round trip নষ্ট করা এড়াতে যা স্পষ্টতই reject হতে যাচ্ছে — কিন্তু এটি advisory, কারণ একজন malicious client শুধু SDK এড়িয়ে যেতে পারে। API gateway shared Redis store-এর বিরুদ্ধে authoritative, global limit কার্যকর করে — এটিই সঠিকতার জন্য আসলে গুরুত্বপূর্ণ। Service mesh sidecar গুলো internal service-to-service limit কার্যকর করে যাতে একটি overloaded internal caller একটি shared internal dependency-কে না খেয়ে ফেলে, যা gateway কখনো দেখে না কারণ সেই traffic edge অতিক্রম করে না। আপনার gateway দরকার; অন্য দুটি হলো defense in depth এবং UX।
