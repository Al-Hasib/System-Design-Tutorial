# Practice & Interview Questions

**১. HTTP-এর জন্য "stateless" হওয়ার অর্থ কী, এবং scalability-এর জন্য এই property কেন গুরুত্বপূর্ণ?**
প্রতিটি request স্বাধীনভাবে handle করা হয়, server-এ আগের requests-এর কোনো memory বহন না করে। এর মানে হলো load balancer-এর পেছনে থাকা যেকোনো server যেকোনো client-এর request handle করতে পারে, যা stateless web servers-এর horizontal scaling-কে সহজ করে তোলে।

**২. PUT এবং PATCH-এর মধ্যে পার্থক্য ব্যাখ্যা করুন।**
PUT প্রদত্ত payload দিয়ে সম্পূর্ণ resource-টি replace করে (idempotent)। PATCH একটি partial update প্রয়োগ করে, শুধুমাত্র নির্দিষ্ট fields modify করে, এবং সাধারণত idempotent হওয়ার নিশ্চয়তা দেয় না।

**৩. একটি client দ্বারা retry হতে পারে এমন APIs design করার সময় idempotency কেন গুরুত্বপূর্ণ?**
যদি একটি network call timeout হয়, client হয়তো জানে না server এটি process করেছে কিনা, তাই এটি প্রায়ই retry করে। Idempotent operations (GET, PUT, DELETE) অন্ধভাবে retry করা safe। POST-এর মতো non-idempotent operations retry-তে duplicates তৈরি করতে পারে, যদি না একটি idempotency key-এর মতো কিছু দিয়ে সুরক্ষিত থাকে।

**৪. একটি 401 এবং একটি 403 status code-এর মধ্যে পার্থক্য কী?**
401 Unauthorized মানে server জানে না caller কে — authentication অনুপস্থিত বা invalid। 403 Forbidden মানে server জানে caller কে, কিন্তু তার এই action করার permission নেই।

**৫. HTTPS, HTTP-এর উপর কী যোগ করে, এবং performance cost কী?**
HTTPS HTTP-কে TLS-এ মুড়ে দেয়, যা একটি certificate-এর মাধ্যমে server-কে authenticate করে এবং সব traffic encrypt করে। খরচ হলো TLS handshake, যা প্রথম request পাঠানোর আগে অতিরিক্ত round trips যোগ করে — session resumption, TLS 1.3-এর 1-RTT/0-RTT modes, এবং connection reuse দ্বারা কমানো হয়।

**৬. একটি client `POST /orders` call করে এবং response পাওয়ার আগেই connection drop হয়ে যায়। এটিকে retry-safe করার একটি উপায় design করুন।**
Client-কে একটি unique idempotency key (যেমন, একটি UUID) তৈরি করতে দিন এবং সেটি একটি `Idempotency-Key` header-এ পাঠান। Server সেই value দিয়ে key করা সম্পূর্ণ request results সংরক্ষণ করে; যদি সে একই key আবার দেখে, তাহলে দ্বিতীয় একটি order তৈরি করার বদলে original response ফেরত দেয়।

**৭. একটি architectural style হিসেবে REST-এর core constraints কী কী?**
URLs দ্বারা চিহ্নিত resources, methods-এর একটি uniform set-এর মাধ্যমে manipulation (standard HTTP verbs), statelessness, যেখানে উপযুক্ত সেখানে cacheability, এবং পুরো API জুড়ে একটি consistent/uniform interface।

**৮. একটি blogging platform-এর জন্য REST endpoints design করুন যেখানে users posts লেখেন এবং posts-এর comments থাকে।**
`GET /users/{id}/posts` (একজন user-এর posts list করা), `POST /posts` (একটি post তৈরি করা), `GET /posts/{id}` (একটি পড়া), `PUT /posts/{id}` (replace করা), `DELETE /posts/{id}` (সরানো), `GET /posts/{id}/comments` (comments list করা), `POST /posts/{id}/comments` (একটি comment যোগ করা) — resources হলো nouns, nesting comment-belongs-to-post সম্পর্ক প্রকাশ করে।

**৯. JSON body-তে একটি error message সহ `200 OK` ফেরত দেওয়া কেন খারাপ REST design হিসেবে বিবেচিত হতে পারে?**
এটি uniform interface ভেঙে দেয় — clients (এবং caches, monitoring, এবং load balancers-এর মতো infrastructure) body parse না করে success/failure নির্ধারণ করতে status codes-এর উপর নির্ভর করে। Errors-এর জন্য 200 ফেরত দেওয়া প্রতিটি caller-কে payload পরীক্ষা করতে বাধ্য করে, যা standard HTTP semantics এবং tooling-কে অকার্যকর করে দেয়।

**১০. HTTP/1.1 এবং HTTP/2-এর মধ্যে পার্থক্য কী যা performance-এর জন্য গুরুত্বপূর্ণ?**
Resources parallel-এ fetch করতে HTTP/1.1-এর কার্যকরভাবে একাধিক connections (বা keep-alive সহ serialized requests) দরকার হয়, যা overhead-এর দিকে নিয়ে যায়। HTTP/2 একটি single TCP connection-এর উপর অনেক requests এবং responses multiplex করে এবং headers compress করে, latency এবং connection overhead কমায়, বিশেষত অনেক concurrent calls সহ pages/APIs-এর জন্য।

**১১. GET-এর কোনো side effect না থাকা ("safe" হওয়া) কেন প্রত্যাশিত?**
কারণ browsers, proxies, এবং crawlers স্বয়ংক্রিয়ভাবে GET requests ইস্যু করতে পারে (prefetching, retries, caching) প্রতিটির জন্য user-এর explicit intent ছাড়াই। যদি GET state পরিবর্তন trigger করত, তাহলে সেই স্বয়ংক্রিয় requests অনিচ্ছাকৃত actions ঘটাতে পারত।

**১২. একটি system design interview-এ, একটি public-facing API-এর জন্য একটি custom RPC-style API-এর বদলে REST বেছে নেওয়াকে আপনি কীভাবে justify করবেন?**
REST-এর resource-oriented design ভালোভাবে বোঝা যায় এমন HTTP semantics (methods, status codes, caching) ব্যবহার করে, যা third-party developers-দের জন্য শেখা সহজ করে তোলে, HTTP layer-এ (CDNs, browsers) cache করা সহজ করে তোলে, এবং একটি bespoke RPC protocol-এর তুলনায় predictably evolve/version করা সহজ করে তোলে।
