# Practice ও Interview প্রশ্ন

**১. একটি microservices আর্কিটেকচারে API Gateway কোন মূল সমস্যা সমাধান করে?**
এটি client-দের একটি একক, unified entry point দেয়, তাদের ডজন ডজন স্বাধীন service সম্পর্কে জানতে এবং আলাদাভাবে call করতে হয় না, এবং এটি cross-cutting concern গুলোকে (auth, rate limiting, logging) কেন্দ্রীভূত করে যাতে প্রতিটি service সেগুলো পুনরায় implement না করে।

**২. একটি API Gateway সাধারণত যে মূল দায়িত্বগুলো পালন করে তার মধ্যে অন্তত চারটি তালিকাভুক্ত করুন।**
সঠিক backend service-এ routing, authentication/authorization, rate limiting/throttling, request/response transformation, একাধিক service call-এর aggregation, এবং কেন্দ্রীভূত observability (logging/metrics/tracing) — এর মধ্যে যেকোনো চারটি।

**৩. প্রতিটি microservice-কে নিজস্ব auth check implement করতে দেওয়ার চেয়ে gateway-তে authentication কেন্দ্রীভূত করা সাধারণত কেন নিরাপদ?**
এটি attack surface এবং সম্ভাবনা কমায় যে অনেক স্বাধীনভাবে maintain করা service-এর মধ্যে একটি auth ভুলভাবে বা অসামঞ্জস্যপূর্ণভাবে implement করবে; প্রতিটি request একইভাবে, একবার, কোনো backend service-এ পৌঁছানোর আগেই যাচাই করা হয়।

**৪. একটি API Gateway-র context-এ "aggregation" কী, এবং বিশেষত mobile client-দের জন্য এটি কেন গুরুত্বপূর্ণ?**
Aggregation হলো যখন gateway একটি একক client request-কে একাধিক internal service call-এ fan out করে এবং ফলাফলগুলো একটি response-এ একত্রিত করে। এটি বিশেষত ধীর/high-latency connection-এর mobile client-দের জন্য গুরুত্বপূর্ণ, কারণ এটি client থেকে কয়েকটি ধীর round trip-এর বদলে একটি round trip দেয়, এবং দ্রুত internal network link-এর মাধ্যমে fan-out করে।

**৫. "edge monolith" anti-pattern কী, এবং কীভাবে এটি এড়ানো যায়?**
এটি তখন ঘটে যখন একটি API Gateway সময়ের সাথে সাথে cross-cutting, generic concern-এর মধ্যে সীমাবদ্ধ থাকার বদলে প্রকৃত business logic জমা করে, কার্যত edge-এ সমস্ত service-কে একসাথে coupling করে একটি অনিচ্ছাকৃত monolith হয়ে ওঠে। শৃঙ্খলাবদ্ধ scope রেখে এটি এড়ানো যায়: gateway-কে routing, auth, rate limiting, এবং transformation-এ সীমাবদ্ধ রাখুন, এবং business logic মালিক service গুলোর ভেতরেই রাখুন।

**৬. একটি single generic API যে সমস্যায় হিমশিম খায়, Backend-for-Frontend (BFF) pattern তার কোন সমস্যা সমাধান করে?**
বিভিন্ন client type-এর (mobile, web, partner) খুবই ভিন্ন ভিন্ন চাহিদা আছে — payload size, round trip সংখ্যা, data shape। এদের সবাইকে সন্তুষ্ট করার চেষ্টা করা একটি single generic API সাধারণত optional field ও conditional logic দিয়ে ভারী হয়ে যায়। BFF প্রতিটি client type-কে তার নিজস্ব tailored backend layer দিয়ে এটি সমাধান করে।

**৭. একটি BFF সাধারণত underlying microservices-এর সাথে কীভাবে সম্পর্কিত — এটি কি সেগুলো প্রতিস্থাপন করে?**
না — BFF client এবং underlying microservices-এর মাঝে বসে থাকে, সেগুলোকে call করে এবং একটি নির্দিষ্ট client type-এর জন্য বিশেষভাবে তাদের response shape/aggregate করে। underlying service গুলো নিজেরা সমস্ত BFF জুড়ে generic ও reusable থাকে।

**৮. একটি API Gateway-কে কেন একটি critical single point of failure হিসেবে বিবেচনা করা হয়, এবং সেই ঝুঁকি কীভাবে কমানো হয়?**
কারণ এটি system-এর প্রায় সমস্ত client traffic-এর path-এ বসে থাকে — যদি এটি বন্ধ হয়ে যায়, client-রা কোনো backend service-এ পৌঁছাতে পারবে না যদিও সেই service গুলো সুস্থ থাকে। যেকোনো critical component-এর মতোই এটি কমানো হয়: তাদের নিজস্ব load-balanced, multi-zone setup-এর পেছনে একাধিক redundant gateway instance deploy করে।

**৯. একটি company-র একটি mobile app team এবং একটি web app team প্রায়ই একটি shared "generic API" team-এর জন্য অপেক্ষা করে আটকে থাকে, যাতে তারা প্রতিটির প্রয়োজনীয় field যোগ করতে পারে। BFF pattern এখানে কীভাবে সাহায্য করতে পারে?**
প্রতিটি client team তাদের নিজস্ব BFF-এর মালিক হতে পারে, একটি shared team-কে one-size-fits-all contract নিয়ে negotiate করার জন্য অপেক্ষা না করেই তাদের ঠিক প্রয়োজন অনুযায়ী API shape ও aggregation তৈরি করতে পারে, আর underlying service গুলো তাদের নিজ নিজ service team-এর মালিকানাধীন থাকে।

**১০. একটি API Gateway এবং একটি plain reverse proxy-র মধ্যে ভূমিকার পার্থক্য কী?**
একটি reverse proxy মূলত backend server লুকায় এবং transport/HTTP level-এ routing/TLS termination করে। একটি API Gateway এর উপর ভিত্তি করে তৈরি কিন্তু application-aware, API-specific সক্ষমতা যোগ করে: authentication/authorization, প্রতি client rate limiting, request/response transformation, এবং multi-service aggregation।

**১১. legacy এবং modern service-এর মিশ্রণ থাকা একটি system-এ gateway-তে request/response transformation কেন উপযোগী হতে পারে?**
Gateway একটি external REST call-কে যেকোনো protocol-এ অনুবাদ করতে পারে যা একটি নির্দিষ্ট backend internally আসলে ব্যবহার করে (যেমন, একটি legacy service-এর জন্য gRPC বা SOAP), যা external client-দের একটি সামঞ্জস্যপূর্ণ modern interface ব্যবহার করতে দেয় প্রতিটি backend service-কে সরাসরি এটি সমর্থন করতে হয় না।

**১২. একটি interview-এ, আপনি কীভাবে সিদ্ধান্ত নেবেন যে একটি system-এর একটি plain API Gateway প্রয়োজন, নাকি প্রতিটি client-এর জন্য আলাদা BFF সহ একটি Gateway?**
যদি সমস্ত client-এর প্রায় একই data shape এবং access pattern প্রয়োজন হয়, তাহলে একটি single gateway (সম্ভবত হালকা প্রতি-route customization সহ) সাধারণত যথেষ্ট। যদি client type-গুলোর অর্থপূর্ণভাবে ভিন্ন সীমাবদ্ধতা থাকে — যেমন, একটি bandwidth-সীমিত mobile app বনাম একটি data-সমৃদ্ধ admin dashboard — gateway-র পেছনে আলাদা BFF যুক্তিসঙ্গত, যা কিছু duplication-এর বিনিময়ে প্রতিটি client team-এর autonomy এবং প্রতিটি consumer-এর জন্য একটি ভালো-মানানসই API দেয়।
