# API Gateway ও Backend-for-Frontend Pattern

**কঠিনতা:** Intermediate

## Learning Objectives

- একটি microservices আর্কিটেকচারে API Gateway কোন সমস্যার সমাধান করে তা ব্যাখ্যা করা।
- একটি API Gateway-র মূল দায়িত্বগুলো তালিকাভুক্ত করা: routing, auth, rate limiting, aggregation, transformation।
- Backend-for-Frontend (BFF) pattern বর্ণনা করা এবং কেন বিভিন্ন client-এর ভিন্ন ভিন্ন API shape প্রয়োজন হতে পারে তা ব্যাখ্যা করা।
- একটি সাধারণ reverse proxy, একটি API Gateway, এবং একটি BFF-এর মধ্যে তুলনা করা।
- একটি API Gateway যে tradeoffs এবং failure mode গুলো নিয়ে আসে তা চিহ্নিত করা (single point of failure, বাড়তি latency)।

## Script

### Hook / Intro

আগের ভিডিওতে আমরা দেখেছিলাম যে একটি reverse proxy আপনার server-গুলোর সামনে বসে সেগুলোকে client থেকে আড়াল করে। এখন কল্পনা করুন আপনার একটিমাত্র backend service নেই — আপনার আছে ত্রিশটি। microservices-এর উপর তৈরি একটি আধুনিক application-এ থাকতে পারে একটি user service, একটি order service, একটি payments service, একটি recommendations service, এবং আরও অনেক, যেগুলোর প্রতিটি স্বাধীনভাবে deploy করা যায়। যদি একটি mobile app-কে ত্রিশটি service-এরই ঠিকানা জানতে হতো, প্রতিটির সাথে আলাদাভাবে authenticate করতে হতো, এবং প্রতিটির নিজস্ব quirks আলাদাভাবে সামলাতে হতো, তাহলে সেটি তৈরি করা একটি দুঃস্বপ্ন হতো এবং পরিবর্তন করা তার চেয়েও বড় দুঃস্বপ্ন হতো। এটিই ঠিক সেই সমস্যা যা API Gateway সমাধান করে — গোটা microservices আর্কিটেকচারের জন্য একটি একক, বুদ্ধিমান front door। এবং একবার আমরা gateway বুঝে গেলে, আমরা দেখব এমন একটি pattern যা প্রায়ই এদের পাশাপাশি থাকে: Backend-for-Frontend, বা BFF।

### সমস্যাটি: অনেক Service, অনেক Client

চলুন পরিস্থিতিটি ভালোভাবে বুঝি। একটি microservices আর্কিটেকচারে, প্রতিটি service সাধারণত নিজের data-র মালিক এবং নিজের API expose করে, এবং service-গুলো স্বাধীনভাবে deploy ও scale করা হয় — এটি team autonomy এবং scalability-র জন্য চমৎকার, যা আমরা Module 7-এ বিস্তারিতভাবে খতিয়ে দেখব। কিন্তু এটি edge-এ একটি coordination সমস্যা তৈরি করে: প্রতিটি client — একটি web app, একটি iOS app, একটি Android app, একটি third-party partner integration — এখন একটি single screen বানাতে সম্ভবত অনেকগুলো ভিন্ন service-এর সাথে কথা বলতে হতে পারে। আরও খারাপ, প্রতিটি service-কেই cross-cutting concern গুলো ডুপ্লিকেট করতে হবে: authentication, rate limiting, logging, request validation। এটি একইসাথে অপচয়মূলক এবং বিপজ্জনক — ত্রিশটি স্বাধীনভাবে maintain করা service জুড়ে অসামঞ্জস্যপূর্ণ auth enforcement একটি অপেক্ষমাণ security incident।

### একটি API Gateway কী করে

একটি API Gateway হলো একটি একক entry point যা আপনার সমস্ত backend microservices-এর সামনে বসে থাকে, এবং এটি ঠিক সেই cross-cutting concern গুলোকে কেন্দ্রীভূত করে যাতে পৃথক service-গুলোকে সেগুলো পুনরায় implement করতে না হয়। চলুন এর মূল কাজগুলো একে একে দেখি।

**Routing।** Gateway প্রতিটি আগত request গ্রহণ করে এবং URL path, header, বা অন্যান্য নিয়মের ভিত্তিতে সঠিক backend service-এ এটি route করে — `/users/*` যায় user service-এ, `/orders/*` যায় order service-এ। এটিই সেই reverse-proxy foundation যা আমরা গত ভিডিওতে কভার করেছি।

**Authentication এবং authorization।** প্রতিটি microservice আলাদাভাবে একটি JWT বা session token যাচাই করার বদলে, gateway এটি একবার, কেন্দ্রীয়ভাবে করে, এবং হয় অননুমোদিত request গুলো সাথে সাথে প্রত্যাখ্যান করে অথবা একটি verified identity downstream-এ পাঠিয়ে দেয়। এটি attack surface এবং fleet-এর কোথাও একটি অসামঞ্জস্যপূর্ণ security implementation হওয়ার সম্ভাবনা নাটকীয়ভাবে কমিয়ে দেয়।

**Rate limiting এবং throttling।** Gateway নির্দিষ্ট করতে পারে একটি নির্দিষ্ট client বা API key কতগুলো request করার অনুমতিপ্রাপ্ত, যা backend service গুলোকে সেই traffic পৌঁছানোর আগেই — একটি abusive client দ্বারা হোক বা নিছক organic traffic spike দ্বারা হোক — overwhelmed হওয়া থেকে রক্ষা করে। rate limiting-এর পেছনের প্রকৃত algorithm গুলো নিয়ে আমরা এই course-এর পরের দিকে বিস্তারিত আলোচনা করব।

**Request/response transformation।** Gateway request এবং response গুলোকে পুনর্গঠন করতে পারে — protocol রূপান্তর করা (যেমন, edge-এ একটি REST call-কে internally একটি gRPC call-এ অনুবাদ করা), header যোগ বা বাদ দেওয়া, বা payload পুনর্বিন্যাস করা — যাতে internal service এবং external client-গুলোর অভিন্ন format-এ একমত হতে না হয়।

**Aggregation।** সম্ভবত সবচেয়ে শক্তিশালী ক্ষমতা: একটি একক client request কে gateway একাধিক ভিন্ন service-এর কাছে একাধিক internal call-এ fan out করতে পারে, এবং ফলাফলগুলো একত্রিত করে একটি response-এ পরিণত করতে পারে। একটি সম্ভাব্য ধীর গতির cellular connection-এর মাধ্যমে পাঁচটি আলাদা round trip করার বদলে, mobile app gateway-তে একটি request করে, এবং gateway দ্রুত, low-latency internal network link-এর মাধ্যমে internally fan-out টি সম্পন্ন করে।

**Observability।** Gateway-তে কেন্দ্রীভূত logging, metrics, এবং tracing আপনাকে আপনার গোটা service fleet জুড়ে traffic monitor করার জন্য একটি একক জায়গা দেয়।

জনপ্রিয় বাস্তব-জীবনের বাস্তবায়নের মধ্যে রয়েছে Kong, Amazon API Gateway, Apigee, এবং Netflix-এর Zuul — এবং অনেক team NGINX-এর মতো একটি general-purpose reverse proxy বা একটি Envoy-based service mesh edge-এর মধ্যেও gateway সক্ষমতা তৈরি করে।

### Tradeoffs

এখানে কিছুই বিনামূল্যে পাওয়া যায় না। একটি API Gateway একটি critical, high-traffic component হয়ে ওঠে — যদি এটি বন্ধ হয়ে যায়, সম্ভবত আপনার গোটা system অপ্রবেশযোগ্য হয়ে যাবে, তাই load balancer-এর জন্য আমরা যে redundancy principle গুলোর কথা আলোচনা করেছিলাম, সেই একই principle মেনে এটি তৈরি করতে হবে। এটি প্রতিটি request-এ একটি network hop এবং processing overhead যোগ করে, অর্থাৎ বাড়তি latency যোগ হয়। এবং আপনি যদি সতর্ক না থাকেন, একটি gateway সময়ের সাথে সাথে অতিরিক্ত business logic জমা করতে পারে, যা আপনার microservices-এর edge-এ একটি অনিচ্ছাকৃত monolith-এ পরিণত হয় — একটি classic anti-pattern। শৃঙ্খলা হলো gateway logic-কে cross-cutting, generic concern-এর মধ্যে সীমাবদ্ধ রাখা এবং প্রকৃত business logic service-গুলোর ভেতরেই রাখা।

### Backend-for-Frontend (BFF) Pattern

এখানে একটি সম্পর্কিত কিন্তু ভিন্ন ধারণা আছে। বিভিন্ন client type-এর প্রায়শই একই backend সক্ষমতা থেকে খুবই ভিন্ন চাহিদা থাকে। একটি ধীর, high-latency cellular connection-এর mobile app bandwidth ও battery সংরক্ষণের জন্য অল্প সংখ্যক highly aggregated, minimal-payload response চায়। একটি fast connection-এর desktop web app হয়তো আরও granular data, আরও call, এবং একটি সমৃদ্ধ payload চাইতে পারে যা এটি নমনীয়ভাবে render করতে পারে। একটি third-party partner API-র হয়তো এই দুটোর যেকোনোটি থেকে সম্পূর্ণ ভিন্ন একটি data shape এবং authentication scheme প্রয়োজন হতে পারে।

একটিমাত্র generic gateway API দিয়ে এই সমস্ত চাহিদা মেটানোর চেষ্টা করলে প্রায়শই বিশাল endpoint তৈরি হয়, যেগুলো প্রতিটি client-কে সেবা দেওয়ার জন্য optional field ও conditional logic দিয়ে ঠাসা থাকে — যা maintain করা এক বিশৃঙ্খলা। Backend-for-Frontend pattern *প্রতিটি client type অনুযায়ী* একটি dedicated, পাতলা backend layer তৈরি করে এই সমস্যার সমাধান করে — একটি mobile BFF, একটি web BFF, একটি partner BFF — প্রতিটি সেই নির্দিষ্ট client-এর ঠিক যা প্রয়োজন তার সাথে মানানসই করে তৈরি, যা সেই consumer-এর জন্য আদর্শ উপায়ে underlying service গুলোর call aggregate ও shape করে, আর underlying microservices গুলো নিজেরা generic ও reusable থাকে। প্রতিটি BFF সাধারণত সেই team-এর মালিকানাধীন থাকে যে team সেই client experience-এর মালিকানা রাখে, যা team autonomy-ও বাড়ায় — mobile team একটি shared, generic gateway team-এর জন্য অপেক্ষা না করেই তাদের BFF নিয়ে iterate করতে পারে।

এই দুটোর মধ্যে সম্পর্ক: একটি API Gateway প্রায়ই এখনও সবচেয়ে বাইরের entry point হিসেবে থাকে যা TLS এবং top-level auth-এর মতো universal concern গুলো handle করে, এবং এটি traffic-কে উপযুক্ত BFF-এ route করতে পারে, যেটি তারপর প্রকৃত backend service গুলোকে call করার আগে client-specific aggregation ও shaping করে।

### বাস্তব-জীবনের উদাহরণ

একটি social media app-এর "profile page" screen-এর কথা ভাবুন। Mobile-এ, সেই screen-এর একটি condensed user summary, অল্প সংখ্যক সাম্প্রতিক post, এবং follower/following count প্রয়োজন — সবকিছু একটি lightweight payload-এ, যাতে ফোনে দ্রুত load হয়। Desktop web-এ, একই screen হয়তো একটি সমৃদ্ধ activity feed, বিস্তারিত analytics, এবং আরও embedded media দেখাতে পারে। এর নেপথ্যে, উভয়েরই একটি user service, একটি posts service, এবং একটি social-graph service থেকে data প্রয়োজন। mobile এবং web client প্রতিটি নিজে নিজে তিনটি আলাদা call করে এবং নিজেরা ফলাফল একত্রিত করার বদলে, একটি mobile BFF সেই তিনটি call internally করে এবং একটি tailored, minimal JSON blob রিটার্ন করে, আর একটি আলাদা web BFF একই তিনটি call করে কিন্তু একটি বড় screen এবং দ্রুত connection-এর উপযোগী আরও সমৃদ্ধ, বিস্তারিত payload রিটার্ন করে। উভয় BFF-ই একটি shared API Gateway-র পেছনে থাকে যা প্রতিটি request-এর জন্য — সেটি যে client থেকেই আসুক না কেন — প্রাথমিক auth check এবং TLS termination handle করে।

### Recap

একটি API Gateway হলো microservices আর্কিটেকচারের একক front door, যা routing, authentication, rate limiting, transformation, aggregation, এবং observability-কে কেন্দ্রীভূত করে, যাতে পৃথক service গুলো এগুলো পুনরায় implement না করে। এটিকে redundant ভাবে তৈরি করতে হবে যেহেতু এটি গোটা system-এর জন্য একটি critical dependency হয়ে ওঠে, এবং এর প্রকৃত business logic শোষণ না করার ব্যাপারে শৃঙ্খলাবদ্ধ থাকা উচিত। Backend-for-Frontend pattern প্রতিটি স্বতন্ত্র client type-কে তার নিজস্ব tailored, পাতলা backend layer দিয়ে এটিকে পরিপূরক করে, যাতে one-size-fits-all API response খুবই ভিন্ন consumer-দের জন্য একটি bottleneck হয়ে না ওঠে।

### এরপর কী

আমরা এখন সম্পূর্ণ request/response path কভার করেছি: language হিসেবে HTTP, traffic বিতরণ ও রক্ষাকারী load balancer এবং proxy, এবং edge-এ microservices orchestrate করা gateway/BFF। কিন্তু সবকিছু একটি সাধারণ request-response model-এ খাপ খায় না — কিছু feature-এর জন্য server-এর ঠিক যে মুহূর্তে কিছু ঘটে সেই মুহূর্তেই client-এ data push করা প্রয়োজন। পরের ভিডিওতে, আমরা WebSockets, long polling, এবং Server-Sent Events কভার করব — real-time feature তৈরির তিনটি প্রধান technique, এবং এগুলোর মধ্যে কীভাবে বেছে নিতে হয়।

## Key Takeaways

- একটি API Gateway হলো microservices আর্কিটেকচারের একক entry point, যা routing, auth, rate limiting, transformation, aggregation, এবং observability-কে কেন্দ্রীভূত করে।
- Gateway-তে cross-cutting concern গুলো কেন্দ্রীভূত করলে প্রতিটি microservice-এর সেগুলো স্বাধীনভাবে পুনরায় implement (এবং সম্ভবত ভুলভাবে implement) করা এড়ানো যায়।
- একটি API Gateway একটি critical, high-traffic component যাকে অবশ্যই redundant করতে হবে, এবং সময়ের সাথে সাথে প্রকৃত business logic শোষণ করা এড়ানো উচিত।
- Backend-for-Frontend (BFF) pattern প্রতিটি client type-কে (mobile, web, partner) তার নিজস্ব tailored backend layer দেয়, যা সবাইকে সেবা দেওয়ার চেষ্টা করা একটি বিশাল generic API এড়ায়।
- একটি gateway এবং BFF প্রায়ই একসাথে কাজ করে: gateway একেবারে edge-এ universal concern গুলো handle করে, তারপর aggregation ও shaping-এর জন্য উপযুক্ত client-specific BFF-এ route করে।
