# কেন এই Topic গুরুত্বপূর্ণ: HTTP/HTTPS & REST APIs

> **এক বাক্যে:** HTTP হলো সেই contract যা ভিন্ন ভিন্ন ভাষায়, ভিন্ন ভিন্ন মহাদেশে, অচেনা মানুষদের লেখা code-কে interoperate করতে দেয় — এবং REST হলো conventions-এর সেই set যা সেই contract-কে 200টি অদ্ভুত, undocumented rules-এ পরিণত হওয়া থেকে আটকায়।

## এই ধারণার আগের পৃথিবী

কল্পনা করুন প্রতিটি API নিজের নিয়ম নিজে তৈরি করছে। একটি service HTTP 200 সহ `{"ok": false}` ফেরত দিয়ে errors সংকেত দেয়। আরেকটি "user not found"-এর জন্য HTTP 500 ফেরত দেয়। একটি `POST /getUser` ব্যবহার করে, আরেকটি `GET /user/delete?id=5`। Caching অসম্ভব কারণ কোনো response reuse করা safe কিনা তা নির্দেশ করার কিছুই নেই। Retries বিপজ্জনক কারণ কোনো operation repeat করা safe কিনা তা নির্দেশ করার কিছুই নেই।

এটি কাল্পনিক নয় — internal APIs-এর একটি বড় অংশ আসলে এরকমই দেখতে হয়। এর মূল্য প্রতিদিন দিতে হয়: প্রতিটি integration-এর জন্য custom code দরকার, প্রতিটি client-এর জন্য custom error handling দরকার, এবং generic infrastructure (proxies, CDNs, gateways, monitoring) আপনাকে সাহায্য করতে পারে না কারণ এটি আপনার traffic বুঝতে পারে না।

## এটি যেসব সমস্যার সমাধান করে

### ১. Infrastructure যা তার কাজ করতে পারে না
**আপনি যা দেখেন:** আপনার CDN কিছুই cache করে না। আপনার load balancer safely retry করতে পারে না। আপনার monitoring errors থেকে successes আলাদা করতে পারে না।

**কেন এমন হয়:** request path-এ থাকা shared infrastructure-এর প্রতিটি অংশ HTTP semantics-এর ভিত্তিতে সিদ্ধান্ত নেয় — method, status code, cache headers। যদি একটি write `GET` হিসেবে পাঠানো হয় বা একটি error `200` ফেরত দেয়, তাহলে infrastructure-কে মিথ্যা বলা হচ্ছে, এবং সে অনুযায়ীই সে আচরণ করে।

**HTTP semantics কীভাবে এটি সমাধান করে:** `GET` safe এবং cacheable, তাই proxies এবং CDNs এটি cache করতে পারে। `PUT` এবং `DELETE` idempotent, তাই একটি load balancer বা client timeout-এর পরে কাজ duplicate না করে সেগুলো retry করতে পারে। `POST` কোনোটিই নয়, তাই কিছুই এটি অন্ধভাবে retry করে না। Status codes প্রতিটি intermediary-কে একটি সংখ্যায় বলে দেয় কী ঘটেছে। এই semantics অনুসরণ করাই caching, retries, এবং monitoring *বিনামূল্যে* পাওয়ার উপায়, নিজে থেকে সেগুলো তৈরি করার বদলে।

### ২. Retries যা নীরবে duplicates তৈরি করে
**আপনি যা দেখেন:** একজন user-কে তিনবার charge করা হয়েছে। Client timeout হয়ে retry করেছিল; আসলে server-side-এ তিনটি request-ই সফল হয়েছিল।

**কেন এমন হয়:** operation-টি idempotent ছিল না, এবং protocol-এ এমন কিছু ছিল না যা এটা নির্দেশ করে। Timeouts অস্পষ্ট — client "কখনো পৌঁছায়নি" এবং "সফল হয়েছে কিন্তু response হারিয়ে গেছে"-এর মধ্যে পার্থক্য করতে পারে না।

**এই topic কীভাবে এটি সমাধান করে:** কোন methods idempotent তা বোঝা, এবং একটি idempotency key দিয়ে কীভাবে non-idempotent operations-কে idempotent করা যায় তা জানা, retries-কে safe করে তোলে। একটি distributed system-এ retries ঐচ্ছিক নয়, তাই এটি কোনো ছোট বিষয় নয়।

### ৩. API designs যা ব্যবহার করতে একটি meeting দরকার হয়
**আপনি যা দেখেন:** প্রতিটি নতুন client integration-এ এক সপ্তাহ এবং একটি Slack thread লাগে, কারণ API-এর shape অনুমান করার বদলে ব্যাখ্যা করতে হয়।

**কেন এমন হয়:** Conventions ছাড়া, API একটি team-এর অভ্যাস encode করে। Resource naming, error format, pagination, এবং filtering — সবকিছুই প্রতিবার স্থানীয়ভাবে এবং ভিন্নভাবে উদ্ভাবিত হয়।

**REST কীভাবে এটি সমাধান করে:** Resource-oriented URLs, standard methods, এবং standard status codes মানে হলো একজন যোগ্য engineer docs না পড়েই আপনার বেশিরভাগ API সঠিকভাবে অনুমান করতে পারবেন। সেই predictability-ই REST-এর প্রকৃত মূল্য — architectural purity নয়।

### ৪. Credentials এবং data যা path-এ থাকা যে কেউ পড়তে পারে
**আপনি যা দেখেন:** public Wi-Fi-তে ধরা পড়া একটি session token একজন user-কে impersonate করতে replay করা হয়।

**কেন এমন হয়:** Plain HTTP clear text-এ প্রেরিত হয়। user এবং আপনার server-এর মধ্যে থাকা প্রতিটি router, Wi-Fi access point, এবং ISP এটি পড়তে এবং modify করতে পারে।

**HTTPS কীভাবে এটি সমাধান করে:** TLS প্রদান করে encryption (কেউ এটি পড়তে পারবে না), integrity (কেউ এটি অলক্ষ্যে modify করতে পারবে না), এবং authentication (client verify করতে পারে সে সত্যিই আপনার server-এর সাথে কথা বলছে, কোনো impostor-এর সাথে নয়)। এই কারণেই HTTPS এখন HTTP/2, service workers, geolocation, এবং বেশিরভাগ modern browser APIs-এর জন্য একটি কঠোর prerequisite।

## আপনি যে মূল্য দেন

- **REST কিছু সমস্যার জন্য একটি খারাপ fit।** অত্যন্ত relational client needs over-fetching এবং under-fetching-এর কারণ হয় (যে সমস্যাটি সমাধান করতে GraphQL-এর অস্তিত্ব)। Real-time push request/response-এর সাথে একদমই খাপ খায় না (তাই WebSockets এবং SSE)। High-throughput internal service-to-service calls text headers এবং JSON parsing-এ real overhead দেয় (তাই gRPC)।
- **Purity নিয়ে বিতর্ক সময় নষ্ট করে।** একটি endpoint "সত্যিকারভাবে RESTful" কিনা তা নিয়ে তর্ক খুব কমই কোনো product-কে উন্নত করে। Orthodoxy-র চেয়ে consistency অনেক বেশি গুরুত্বপূর্ণ।
- **HTTPS-এর একটি খরচ আছে।** handshake-এর জন্য অতিরিক্ত round trips, encryption-এর জন্য CPU, এবং certificate lifecycle management। সবই ছোট এবং তার মূল্য আছে, কিন্তু শূন্য নয় — যে কারণে connection reuse এবং session resumption গুরুত্বপূর্ণ।

## কখন এটি আপনার দরকার — এবং কখন নয়

| REST over HTTP কখন মানানসই | কখন অন্য কিছু বেছে নেবেন |
|---|---|
| Public APIs, বা যা কিছু third parties ব্যবহার করে | আপনার server-push বা bidirectional streams দরকার → WebSockets/SSE |
| CRUD-আকৃতির resources স্বাভাবিকভাবে মানানসই হয় | Clients-দের flexible, nested queries দরকার → GraphQL |
| আপনি চান caching এবং proxies বিনামূল্যে কাজ করুক | Internal, latency-critical, high-volume RPC → gRPC |
| Broad client compatibility গুরুত্বপূর্ণ | আপনি বিশাল binary payloads সরাচ্ছেন |

## কেন এটি Interviews-এ দেখা যায়

প্রায় প্রতিটি system design round-এই API design দেখা যায়, সাধারণত "API-টি দেখতে কেমন হবে?" হিসেবে। Interviewers লক্ষ্য করেন সঠিক method choice, sensible status codes, idempotency সম্পর্কে সচেতনতা, list endpoints-এর জন্য pagination, এবং versioning। `POST` বনাম `PUT` সঠিকভাবে বলা এবং একটি payments endpoint-এর জন্য idempotency keys উল্লেখ করা একটি ছোট বিষয় যা real production experience-এর সংকেত দেয়।

## এটি কীভাবে সংযুক্ত

HTTP semantics-ই পরবর্তী বেশ কিছু topic সম্ভব করে তোলে। **Caching এবং CDNs** নির্ভর করে `GET` safe হওয়া এবং cache-control headers-এর উপর। **Load balancers এবং API gateways** methods, paths, এবং status codes-এর ভিত্তিতে route এবং retry করে। **Idempotency** হলো distributed systems-এ HTTP method semantics-এর একটি সরাসরি সম্প্রসারণ। **gRPC এবং GraphQL** সবচেয়ে ভালোভাবে বোঝা যায় নির্দিষ্ট কিছু জায়গার প্রতিক্রিয়া হিসেবে যেখানে REST over HTTP একটি খারাপ fit।

**পরবর্তী:** [Load Balancing Explained](../07-load-balancing-explained/why.md) — কীভাবে এই সমস্ত requests একাধিক server-এ ছড়িয়ে দেওয়া হয়।
