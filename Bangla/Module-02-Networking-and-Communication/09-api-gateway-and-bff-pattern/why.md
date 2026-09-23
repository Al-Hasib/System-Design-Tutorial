# এই Topic কেন গুরুত্বপূর্ণ: API Gateway ও Backend-for-Frontend

> **এক বাক্যে:** একবার আপনার অনেক service এবং অনেক ধরনের client হয়ে গেলে, প্রতিটি client-কে সরাসরি প্রতিটি service-এর সাথে কথা বলতে দিলে একটি অপরিচালনাযোগ্য mesh তৈরি হয় — gateway সেই mesh-কে একটি front door-এ সংকুচিত করে, এবং BFF প্রতিটি client-কে তার জন্য বিশেষভাবে তৈরি একটি door দেয়।

## এই ধারণার আগের পৃথিবী

আপনি একটি system-কে পনেরোটি service-এ ভাগ করেছেন। এখন একটি mobile app-এর একটি product page render করতে হবে। এটি catalog service, pricing service, inventory service, reviews service, এবং recommendation service-কে call করে — একটি cellular network-এর মাধ্যমে পাঁচটি round trip, প্রতিটি 200 ms, প্রতিটির নিজস্ব auth handling প্রয়োজন, প্রতিটির একটি URL যা app-এ hardcode করা আছে।

তারপর আপনি একটি service-এর নাম পরিবর্তন করেন। প্রতিটি deployed mobile app version ভেঙে যায়, এবং আপনি user-দের upgrade করতে বাধ্য করতে পারবেন না। তারপর আপনি authentication requirement যোগ করেন, এবং পনেরোটি service প্রতিটি সামান্য ভিন্নভাবে token validation implement করে — এবং একটি ভুলভাবে।

## এটি যে সমস্যাগুলো সমাধান করে

### ১. ধীর network-এ chatty client

**যা আপনি দেখেন:** mobile-এ একটি screen load হতে তিন সেকেন্ড সময় নেয়, যদিও প্রতিটি backend call 30 ms-এ ফলাফল দেয়।

**কেন এটি ঘটে:** Latency-তে round trip-এর প্রাধান্য থাকে, server-এর কাজের নয়। একটি mobile network-এ পাঁচটি ধারাবাহিক round trip মানে যেকোনো processing শুরু হওয়ার আগেই 1-2 সেকেন্ড শুধু অপেক্ষা করা।

**কীভাবে একটি gateway/BFF এটি সমাধান করে:** Client একটি request করে। Gateway সেই পাঁচটি service-এর কাছে সমান্তরালভাবে fan out করে — datacenter-এর ভেতরে, যেখানে round trip sub-millisecond — ফলাফল একত্রিত করে, এবং একটি payload রিটার্ন করে। পাঁচটি ধীর round trip একটিতে পরিণত হয়।

### ২. Internal service topology-র সাথে coupled client

**যা আপনি দেখেন:** আপনি একটি service ভাগ, একত্রীকরণ, বা নাম পরিবর্তন করতে পারেন না কারণ বাস্তবে থাকা mobile app-গুলোতে এর ঠিকানা বেক করা আছে।

**কেন এটি ঘটে:** সরাসরি client-থেকে-service call আপনার internal আর্কিটেকচারকে আপনার public contract-এর অংশ বানিয়ে দেয়।

**কীভাবে একটি gateway এটি সমাধান করে:** এটি একটি indirection layer। Gateway যতক্ষণ একই external API উপস্থাপন করতে থাকে, ততক্ষণ internal service গুলো স্বাধীনভাবে পুনর্গঠন করা যায়। এটিই microservices-কে frozen না রেখে বিবর্তনযোগ্য করে তোলে।

### ৩. পনেরোবার implement করা cross-cutting policy

**যা আপনি দেখেন:** Authentication, rate limiting, request logging, API key validation, এবং CORS প্রতিটি service-এ পুনরায় implement করা হয় — অসামঞ্জস্যপূর্ণভাবে, এবং প্রতিটিই একটি সম্ভাব্য ফাঁক।

**কেন এটি ঘটে:** কোনো shared entry point না থাকায়, প্রতিটি service নিজেই নিজের security boundary।

**কীভাবে একটি gateway এটি সমাধান করে:** edge-এ একবার authenticate করুন, তারপর একটি verified identity ভেতরে পাঠান। edge-এ একবার rate limit করুন, যেখানে আপনি একটি client-এর ব্যবহারের সম্পূর্ণ চিত্র দেখতে পান। একটি implementation, audit করার একটি জায়গা, ঠিক করার একটি জায়গা।

### ৪. একটি API যা কোনো client-এর জন্যই ঠিকমতো মানানসই নয়

**যা আপনি দেখেন:** mobile team ছোট, ছাঁটাই করা payload চায়। web team সমৃদ্ধ, nested data চায়। partner API-র সবচেয়ে বেশি প্রয়োজন স্থিতিশীলতা। একটি single shared API এই তিনজনকেই সেবা দেওয়ার জন্য optional field ও query parameter দিয়ে ভারী হয়ে যায়, এবং কাউকেই ভালোভাবে সেবা দেয় না।

**কেন এটি ঘটে:** বিভিন্ন client-এর সত্যিকারভাবেই ভিন্ন চাহিদা আছে — screen size, network quality, update cadence, এবং trust level সবই আলাদা।

**কীভাবে BFF এটি সমাধান করে:** প্রতিটি client type-কে তার নিজস্ব পাতলা backend দিন, যার মালিক সেই client-এর team, এবং যা সেই client ঠিক যা render করে তার সাথে মানানসই। mobile BFF আক্রমণাত্মকভাবে payload ছাঁটতে পারে; web BFF আরও বেশি রিটার্ন করতে পারে; তারা স্বাধীনভাবে, প্রতিটি team-এর নিজস্ব গতিতে বিবর্তিত হয়।

## যে মূল্য আপনাকে দিতে হবে

- **একটি single point of failure এবং একটি bottleneck।** সবকিছু এর মধ্য দিয়ে যায়। এটিকে redundant, autoscaled, এবং সতর্কতার সাথে monitor করা আবশ্যক — এবং এর failure সম্পূর্ণ।
- **এটি নতুন monolith হয়ে উঠতে পারে।** Business logic ধীরে ধীরে gateway-তে ঢুকে পড়ে; শীঘ্রই কিছু ship করতে প্রতিটি team-এর সেখানে একটি change প্রয়োজন হয়, এবং আপনি সেই deployment bottleneck পুনর্নির্মাণ করেছেন যা এড়াতে আপনি service গুলো ভাগ করেছিলেন। শৃঙ্খলাটি হলো: routing, auth, এবং composition — domain logic নয়।
- **BFF কোড বহুগুণ বাড়ায়।** তিনটি client মানে তৈরি, deploy, monitor, এবং সিঙ্কে রাখার জন্য তিনটি BFF। প্রকৃত duplication, শুধুমাত্র তখনই যুক্তিসঙ্গত যখন client-এর চাহিদা সত্যিকারভাবে ভিন্ন হয়।
- **একটি অতিরিক্ত hop।** Latency এবং operational surface উভয়ই বৃদ্ধি পায়, যা তখনই মূল্যবান যখন aggregation hop-এর খরচের চেয়ে বেশি সাশ্রয় করে।

## কখন এটি প্রয়োজন — এবং কখন নয়

| যখন একটি gateway/BFF যোগ করবেন | যখন এটি বাদ দেবেন |
|---|---|
| অনেক service এবং অনেক client type | একটি single service এবং একটি single client |
| Client গুলো high-latency network-এ আছে এবং aggregation প্রয়োজন | সবকিছু একটি datacenter-এর ভেতরে server-to-server |
| আপনার কেন্দ্রীভূত auth, rate limiting, এবং API key প্রয়োজন | একটি plain reverse proxy ইতিমধ্যে আপনার চাহিদা পূরণ করে |
| Internal পরিবর্তন হলেও Public/partner API-কে স্থিতিশীল থাকতে হবে | আপনি প্রাথমিক পর্যায়ে আছেন এবং internal churn-এ কোনো খরচ নেই |

## Interview-এ এটি কেন দেখা যায়

microservices এবং একটি mobile client সম্বলিত যেকোনো design "client কীভাবে এই সমস্ত service-এর সাথে কথা বলে?" এই প্রশ্নটি আমন্ত্রণ জানায়। gateway-র নাম বলা, round trip নির্মূল করতে fan-out aggregation ব্যাখ্যা করা, এবং edge-এ auth রাখা — এটিই প্রত্যাশিত উত্তর। শক্তিশালী candidate-রা না জিজ্ঞেস করেই সতর্কতা যোগ করে: business logic gateway থেকে দূরে রাখুন, এবং এটিকে redundant করুন, কারণ এটি এখন সবকিছুর জন্য critical path-এ আছে।

## এটি কীভাবে সংযুক্ত

Gateway হলো application-level intelligence সহ একটি **reverse proxy**, তাই **rate limiting** প্রয়োগ করা, **TLS** terminate করা, এবং অস্থির downstream service-এর বিরুদ্ধে **circuit breaker** প্রয়োগ করার এটি স্বাভাবিক জায়গা। service গুলো আসলে কোথায় আছে তা জানতে এটি **service discovery**-র উপর নির্ভর করে। **GraphQL** হলো, একভাবে দেখলে, একটি BFF যার aggregation rule গুলো BFF team দ্বারা hardcode করার বদলে query-এর সময় client দ্বারা নির্ধারিত হয়।

**পরবর্তী:** [WebSockets, Long Polling ও Server-Sent Events](../10-websockets-long-polling-and-sse/why.md) — server-কে আগে কথা বলতে হলে কী করবেন।
