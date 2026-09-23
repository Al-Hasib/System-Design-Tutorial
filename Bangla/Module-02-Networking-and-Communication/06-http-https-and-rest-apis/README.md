# HTTP/HTTPS ও REST APIs ব্যাখ্যা করা হলো

**কঠিনতার মাত্রা:** Beginner/Intermediate

## Learning Objectives

- HTTP আসলে কী এবং কেন একে "stateless, request-response" protocol বলা হয় তা ব্যাখ্যা করা।
- HTTPS কীভাবে TLS ব্যবহার করে HTTP-কে সুরক্ষিত করে, এবং system design-এ এটি কেন গুরুত্বপূর্ণ তা বর্ণনা করা।
- প্রমিত HTTP methods, status code ranges, এবং headers চিহ্নিত করা, এবং প্রতিটি কিসের জন্য তা বোঝা।
- REST-কে একটি architectural style হিসেবে ব্যাখ্যা করা এবং একটি resource-oriented API ডিজাইন করা।
- সাধারণ REST API design ভুলগুলো চিহ্নিত করা এবং সেগুলো কীভাবে এড়ানো যায় তা জানা।

## Script

### Hook / Intro

আপনি যখনই একটি webpage load করেন, একটি app-এ "checkout"-এ tap করেন, বা কোনো backend service-এর endpoint-এ hit করেন, তখন আপনি প্রায় নিশ্চিতভাবেই HTTP ব্যবহার করছেন। অতিরঞ্জন ছাড়াই বলা যায়, system design interview এবং real production systems-এ এটিই সবচেয়ে বেশি ব্যবহৃত protocol। তাই আজ আমরা এটিকে সম্পূর্ণভাবে খুলে দেখব: HTTP আসলে কী, HTTPS এর উপর কী যোগ করে, এবং REST কীভাবে আমাদের দুটোর উপর ভিত্তি করে একটি sane এবং consistent উপায়ে API design করার সুযোগ দেয়। এই ভিডিওর শেষে আপনি যেকোনো API দেখলেই বুঝতে পারবেন সেটি "RESTful" কিনা, এবং একটি interview-তে আত্মবিশ্বাসের সাথে ব্যাখ্যা করতে পারবেন কেন একটি `PUT`, `POST` থেকে আলাদা, এবং কেন `https://` শুধু একটি padlock icon-এর চেয়ে বেশি কিছুর জন্য গুরুত্বপূর্ণ।

আগের module-এ আমরা client-server architecture নিয়ে কথা বলেছিলাম — একটি client একটি request পাঠায়, একটি server একটি response ফেরত পাঠায়। HTTP হলো ঠিক সেই নির্দিষ্ট ভাষা যা তারা এই কথোপকথন চালাতে ব্যবহার করে।

### HTTP আসলে কী

HTTP মানে HyperText Transfer Protocol। এটি একটি application-layer protocol — অর্থাৎ এটি TCP/IP-এর উপরে বসে থাকে, যা network জুড়ে bytes-এর প্রকৃত movement পরিচালনা করে, এবং HTTP messages-এর *format* সংজ্ঞায়িত করে: একটি client কীভাবে কিছু চায়, এবং একটি server কীভাবে উত্তর দেয়।

দুটি property HTTP-এর মূল আচরণ নির্ধারণ করে। প্রথমত, এটি request-response: client সবসময় initiate করে, server সবসময় reply করে। plain HTTP-তে server randomly আপনার কাছে data push করে না — এই module-এ পরে আমরা দেখব কীভাবে real-time behavior দরকার হলে WebSockets এই নিয়ম ভাঙে। দ্বিতীয়ত, HTTP stateless। প্রতিটি request সম্পূর্ণভাবে বিচ্ছিন্নভাবে handle করা হয় — server আপনার আগের request মনে রাখে না, যদি না আপনি explicitly state carry করেন, সাধারণত cookies, tokens, বা session identifiers-এর মাধ্যমে যা প্রতিটি request-এর সাথে পাঠানো হয়। এই statelessness আসলে scalability-এর জন্য একটি superpower: কারণ কোনো server-কেই আপনাকে "মনে রাখতে" হয় না, load balancer-এর পেছনে থাকা যেকোনো server যেকোনো request handle করতে পারে, যা horizontal scaling-এর জন্য ঠিক প্রয়োজনীয়।

প্রতিটি HTTP request-এ একটি method, একটি URL, headers, এবং ঐচ্ছিকভাবে একটি body থাকে। প্রতিটি response-এ একটি status code, headers, এবং ঐচ্ছিকভাবে একটি body থাকে। চলুন system design-এ সবচেয়ে গুরুত্বপূর্ণ অংশগুলো নিয়ে আলোচনা করি।

### HTTP Methods

Methods — কখনো কখনো verbs বলা হয় — server-কে বলে দেয় আপনি কী ধরনের action চান:

- `GET` একটি resource retrieve করে। এটি safe হওয়া উচিত (কোনো side effect নেই) এবং idempotent (একে বারবার call করলে একই result পাওয়া যায়)।
- `POST` একটি নতুন resource তৈরি করে বা একটি action trigger করে। Idempotent নয় — একে দুইবার call করলে সাধারণত দুটি জিনিস তৈরি হয়।
- `PUT` একটি resource সম্পূর্ণভাবে replace করে। এটি idempotent — একই PUT পাঁচবার পাঠালে একই final state পাওয়া যায়।
- `PATCH` একটি resource আংশিকভাবে update করে।
- `DELETE` একটি resource সরিয়ে দেয়। Idempotent — যা ইতিমধ্যে চলে গেছে তা delete করলেও ফলাফল হয় "এটি চলে গেছে"।

সেই idempotency পার্থক্যটি সবচেয়ে common interview প্রশ্নগুলোর একটি, কারণ এটি সরাসরি প্রভাবিত করে আপনি একটি distributed system-এ retries কীভাবে handle করেন। যদি একটি client একটি PUT-এর response-এর জন্য অপেক্ষা করতে করতে timeout হয় এবং retry করে, সেটি safe। একটি POST অন্ধভাবে retry করুন, আর আপনি হয়তো একটি duplicate order তৈরি করে ফেলবেন। এই কারণেই real systems retries দরকার হলে POST-এর উপর idempotency keys যোগ করে — একটি concept যা আমরা এই course-এ পরে ভালোভাবে খতিয়ে দেখব।

### Status Codes

Status codes পাঁচটি range-এ ভাগ করা থাকে, এবং range-গুলো ভালোভাবে জানা থাকলে যেকোনো interview-তে আপনাকে sharp মনে হবে:

- **1xx** — informational, সরাসরি খুব একটা দেখা যায় না।
- **2xx** — success। `200 OK`, `201 Created`, `204 No Content`।
- **3xx** — redirection। `301 Moved Permanently`, `304 Not Modified` — সেই শেষটি caching-এর জন্য বিশাল গুরুত্বপূর্ণ, যা আমরা Module 4-এ cover করব।
- **4xx** — client error। `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests`।
- **5xx** — server error। `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`।

লক্ষ্য করুন 401 বনাম 403 অনেককে বিভ্রান্ত করে: 401 মানে "আমি জানি না তুমি কে, দয়া করে authenticate করো"; 403 মানে "আমি জানি তুমি কে, এবং তোমার অনুমতি নেই"। এবং 429 হলো এমন একটি যা এই course-এ পরে rate limiting-এ পৌঁছালে আপনি প্রায়ই দেখবেন।

### Headers, এবং তারপর HTTPS

Headers metadata বহন করে: `Content-Type` receiver-কে বলে দেয় body কীভাবে parse করতে হবে (JSON, HTML, ইত্যাদি), `Authorization` bearer token-এর মতো credentials বহন করে, `Cache-Control` caching behavior নিয়ন্ত্রণ করে, ইত্যাদি। core protocol পরিবর্তন না করেই HTTP কীভাবে flexible থাকে তা এভাবেই সম্ভব হয়।

এখন — HTTPS। HTTPS আসলে TLS, অর্থাৎ Transport Layer Security-এর উপর চলা HTTP। Plain HTTP সবকিছু cleartext-এ পাঠায়: network path-এ থাকা যে কেউ — একজন কফি-শপের Wi-Fi snooper, একটি ISP, একটি malicious router — এটি পড়তে বা এর সাথে ছিনিমিনি করতে পারে। TLS connection-কে encryption দিয়ে মুড়ে দেয় একটি handshake ব্যবহার করে যা তিনটি কাজ করে: এটি certificate-এর মাধ্যমে server-এর identity authenticate করে, asymmetric cryptography ব্যবহার করে একটি shared symmetric encryption key negotiate করে, এবং তারপর সেই session-এর সব HTTP traffic সেই দ্রুত symmetric key দিয়ে encrypt করে। system design-এর ভাষায়, real user data handle করা যেকোনো কিছুর জন্য HTTPS আলোচনার অতীত, এবং এর একটি real performance cost আছে — TLS handshake round trips যোগ করে, যে কারণে TLS session resumption এবং HTTP/2-এর connection multiplexing-এর মতো techniques scale-এ গুরুত্বপূর্ণ হয়ে ওঠে। এটাও জানা ভালো যে HTTP/1.1 প্রায় প্রতি request-এর জন্য একটি connection খোলে (keep-alive সাহায্য করে সাথে), যেখানে HTTP/2 একটি single connection-এর উপর অনেক request multiplex করে, এবং HTTP/3 head-of-line blocking এড়াতে transport-টিকেই UDP-এর উপর QUIC-এ সরিয়ে নেয়। বেশিরভাগ interview-এর জন্য deep protocol internals প্রয়োজন নেই, কিন্তু HTTPS = HTTP + TLS জানা, এবং কেন এটি সামান্য latency-র মূল্য দেয় তা জানা আশা করা হয়।

### REST APIs

REST — Representational State Transfer — HTTP-এর উপর APIs design করার একটি architectural style, যা Roy Fielding তার 2000 সালের dissertation-এ সংজ্ঞায়িত করেছিলেন। এটি কোনো protocol বা standard নয়; এটি constraints-এর একটি set যা একসাথে অনুসরণ করলে APIs তৈরি হয় যা consistent, cacheable, এবং বোঝা সহজ।

মূল ধারণা: সবকিছুই একটি **resource**, যা একটি URL দ্বারা চিহ্নিত, এবং আপনি সেই resource-এর উপর standard HTTP methods ব্যবহার করে কাজ করেন। তাই `/getUserById?id=5` বা `/createNewOrder`-এর মতো একটি endpoint-এর বদলে, REST বলে: resource হলো `/users/5`, এবং action আসে HTTP method থেকে। `GET /users/5` এটি পড়ে। `PUT /users/5` এটি replace করে। `DELETE /users/5` এটি সরিয়ে দেয়। `POST /users` collection-এর অধীনে একটি নতুন তৈরি করে।

জানার মতো অন্যান্য REST principles: এটি stateless হওয়া উচিত (প্রতিটি request self-contained, যা HTTP নিজের সাথেই সামঞ্জস্যপূর্ণ), resources যেখানে যথাযথ সেখানে cacheable হওয়া উচিত, এবং একটি ভালোভাবে design করা REST API সঠিক status codes ব্যবহার করে এবং resources-কে যৌক্তিকভাবে nest করে — `/users/5/orders` user 5-এর অন্তর্গত orders-এর জন্য।

ভালো REST design practices: resource paths-এর জন্য nouns ব্যবহার করুন, verbs নয়; consistently plural resource names ব্যবহার করুন (`/user` নয়, `/users`); filtering, sorting, এবং pagination-এর জন্য query parameters ব্যবহার করুন — `/users?status=active&page=2`; আপনার API version করুন, সাধারণত URL path (`/v1/users`) বা একটি header-এর মাধ্যমে; এবং সবসময় 200 ফেরত দিয়ে body-তে লুকানো একটি error message-এর বদলে অর্থপূর্ণ status codes ফেরত দিন।

### Real-World Example

একটি typical e-commerce checkout flow-এর কথা চিন্তা করুন। `GET /products/123` product details fetch করে — cacheable, safe, idempotent। `POST /cart/items` আপনার cart-এ একটি item যোগ করে — idempotent নয়, কারণ একে দুইবার call করলে দুটি item যোগ হয়। `PUT /cart/items/456` একটি নির্দিষ্ট cart item-এর quantity একটি exact value-তে update করে — idempotent। `POST /orders` অবশেষে order place করে — এবং যেহেতু POST-এর retries বিপজ্জনক, mature systems একটি `Idempotency-Key` header যুক্ত করে যাতে client-এর network drop হয়ে retry করলে, server duplicate টি চিনতে পারে এবং দ্বিতীয় একটি তৈরি করার বদলে original order ফেরত দেয়। লক্ষ্য করুন কীভাবে resource model — products, cart, cart items, orders — স্বাভাবিকভাবেই URLs-এর উপর map হয়, এবং কীভাবে শুধু HTTP method-ই documentation না পড়ে intent বলে দেয়।

### Recap

চলুন সবকিছু একসাথে করি। HTTP একটি stateless, request-response, application-layer protocol যাতে methods, status codes, এবং headers আছে। HTTPS encryption এবং authentication-এর জন্য এটিকে TLS-এ মুড়ে দেয়। REST একটি architectural style যা URLs দ্বারা চিহ্নিত resources-এর উপর CRUD-এর মতো operations map করে, intent প্রকাশ করতে HTTP methods ব্যবহার করে। idempotency, status code ranges, এবং পরিষ্কার resource naming নিয়ে স্বাচ্ছন্দ্য বোধ করুন — এগুলো interview এবং real production API design উভয় ক্ষেত্রেই বারবার আসে।

### What's Next

এখন যেহেতু আমরা জানি একটি single client কীভাবে একটি single server-এর সাথে কথা বলে, স্বাভাবিক পরবর্তী প্রশ্ন হলো: একটি server যথেষ্ট না হলে কী হয়? পরবর্তী ভিডিওতে, আমরা load balancing নিয়ে আলোচনা করব — কীভাবে traffic বুদ্ধিমত্তার সাথে অনেক server-এ ছড়িয়ে দেওয়া হয়, এর পেছনের algorithms, এবং Layer 4 ও Layer 7 load balancing-এর মধ্যে গুরুত্বপূর্ণ পার্থক্য।

## Key Takeaways

- HTTP stateless এবং request-response; statelessness-ই হলো যা load balancer-এর পেছনে horizontal scaling সম্ভব করে তোলে।
- Retries-এর জন্য idempotency গুরুত্বপূর্ণ: GET, PUT, DELETE idempotent; POST এবং PATCH সাধারণত নয়।
- Status code ranges: 2xx success, 3xx redirect, 4xx client error, 5xx server error — 401 বনাম 403 এবং 429 জানুন।
- HTTPS = HTTP + TLS: এটি server-কে authenticate করে, traffic encrypt করে, এবং handshake latency যোগ করে যা connection reuse এবং HTTP/2+ কমাতে সাহায্য করে।
- REST সবকিছুকে URLs দ্বারা addressed resources হিসেবে model করে, action প্রকাশ করতে HTTP methods ব্যবহার করে — path-এ baked-in verbs নয়।
