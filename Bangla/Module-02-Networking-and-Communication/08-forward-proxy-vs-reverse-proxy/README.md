# Forward Proxy vs Reverse Proxy

**Difficulty:** Intermediate

## Learning Objectives

- proxy আসলে কী এবং client ও server-এর মাঝে একটি intermediary কেন উপকারী তা সংজ্ঞায়িত করা।
- forward proxy কীভাবে কাজ করে এবং এটি কার হয়ে কাজ করে তা ব্যাখ্যা করা।
- reverse proxy কীভাবে কাজ করে এবং এটি কার হয়ে কাজ করে তা ব্যাখ্যা করা।
- forward এবং reverse proxy-কে পাশাপাশি তুলনা করা, যেখানে প্রতিটির বাস্তবায়নকারী প্রকৃত product-ও অন্তর্ভুক্ত থাকবে।
- একটি reverse proxy কীভাবে load balancer-এর সাথে সম্পর্কিত এবং কীভাবে আলাদা তা বোঝা।

## Script

### Hook / Intro

আগের ভিডিওতে আমরা load balancer নিয়ে কথা বলেছিলাম, যা server pool-এর সামনে বসে সিদ্ধান্ত নেয় প্রতিটি request কোন backend সামলাবে। সেটা আসলে একটা বিস্তৃত component category-র একটি নির্দিষ্ট কাজ মাত্র: proxy। যখনই কোনো কিছু client এবং server-এর মাঝখানে বসে কারও পক্ষে traffic relay করে, সেটাই proxy। কিন্তু এখানে একটা মোড় আছে যা অনেককে শুরুতে বিভ্রান্ত করে — মূলত দুই ধরনের সম্পূর্ণ ভিন্ন proxy আছে, এবং তারা সম্পূর্ণ ভিন্ন দুই পক্ষকে রক্ষা করার জন্য অস্তিত্বে আছে। একটি forward proxy client-এর হয়ে কাজ করে। একটি reverse proxy server-এর হয়ে কাজ করে। এই একটা বাক্য মাথায় গেঁথে নিলে, এই ভিডিওর বাকি সবকিছু নিজে থেকেই পরিষ্কার হয়ে যাবে।

### Proxy আসলে কী, সাধারণভাবে

একটি proxy হলো একটি intermediary যা client এবং server-এর মাঝখানে বসে তাদের হয়ে request এবং response forward করে। client সরাসরি destination server-এর সাথে কথা না বলে, proxy-র সাথে কথা বলে, এবং proxy destination-এর সাথে কথা বলে — অথবা উল্টোটাও হতে পারে। যেহেতু proxy সেই কথোপকথনের মাঝখানে বসে থাকে, তাই এটি শুধু byte forward করার চেয়ে আরও অনেক কিছু করতে পারে: এটি response cache করতে পারে, request filter বা block করতে পারে, traffic log করতে পারে, content rewrite করতে পারে, security যোগ করতে পারে, এবং যে পক্ষকে এটি represent করছে তার identity লুকাতে পারে।

যে মূল প্রশ্নটি আপনাকে বলে দেয় আপনি কোন ধরনের proxy নিয়ে কাজ করছেন, তা সহজ: **এটি কার identity লুকাচ্ছে, এবং কার স্বার্থ রক্ষা করছে?**

### Forward Proxy

একটি forward proxy একদল client-এর সামনে বসে থাকে, এবং server-এর দৃষ্টিকোণ থেকে মনে হয় traffic proxy থেকে আসছে, প্রকৃত client থেকে নয়। client নিজে থেকেই configure করা থাকে যাতে তার traffic forward proxy-র মধ্য দিয়ে পাঠানো হয়।

একটা corporate office network-এর কথা ভাবুন। প্রতিটি কর্মীর laptop এমনভাবে configure করা থাকে যাতে সব outbound web traffic কোম্পানির proxy server-এর মধ্য দিয়ে route হয়। কেউ যখন কোনো website visit করে, destination server শুধু proxy-র IP address-ই দেখতে পায় — এটির কোনো ধারণা নেই ঠিক কোন কর্মী request পাঠিয়েছে। কোম্পানি এটি অনেক কারণে ব্যবহার করে: content filtering (নির্দিষ্ট category-র site-এ প্রবেশ block করা), compliance-এর জন্য কর্মীদের traffic monitor ও log করা, প্রায়শই ব্যবহৃত resource cache করা যাতে একাধিক কর্মী একই content বারবার redundant-ভাবে re-download না করে, এবং নিরাপত্তার জন্য বাইরের দুনিয়া থেকে internal IP address ও network topology লুকানো।

আরেকটা সাধারণ উদাহরণ হলো VPN বা geographic restriction এড়ানোর জন্য ব্যবহৃত কোনো public web proxy-র মতো service — আপনার traffic proxy-র location থেকে বের হয়ে যায়, তাই destination server মনে করে request যেখান থেকেই proxy আছে সেখান থেকে আসছে, আপনার কাছ থেকে নয়। মূল বৈশিষ্ট্য: **একটি forward proxy client-কে রক্ষা করে এবং represent করে**, client-কে server থেকে লুকিয়ে রাখে।

### Reverse Proxy

একটি reverse proxy ঠিক তার বিপরীত কাজ করে: এটি একদল server-এর সামনে বসে থাকে, এবং client-এর দৃষ্টিকোণ থেকে মনে হয় reverse proxy-ই *হলো* server। client-এর কোনো ধারণা নেই, এবং জানার প্রয়োজনও নেই, যে এর পেছনে একটি সম্পূর্ণ backend server-এর fleet রয়েছে।

প্রায় প্রতিটি production web architecture-ই এই pattern ব্যবহার করে। আপনি যখন কোনো website-এ যান, আপনি প্রায়ই সরাসরি application server-এর সাথে কথা বলছেন না — আপনি NGINX, HAProxy, বা কোনো cloud load balancer-এর মতো একটি reverse proxy-র সাথে কথা বলছেন, যা তারপর আপনার request-কে internally অনেকগুলো backend server-এর একটির কাছে forward করে। Reverse proxy-গুলো server-পক্ষের সুবিধার জন্যই থাকে: তারা TLS terminate করে যাতে প্রতিটি backend server-কে আলাদাভাবে certificate management করতে না হয়, তারা edge-এর কাছে static বা প্রায়শই request করা content cache করতে পারে, তারা response compress করে, তারা বাইরের দুনিয়া থেকে internal network topology ও প্রকৃত server IP লুকিয়ে রাখে (একটি বাস্তব security সুবিধা — attacker-রা এমন backend server-কে সরাসরি target করতে পারে না যা তারা দেখতেই পায় না), এবং — এখানেই overlap-এর জায়গা — তারা প্রায়ই তাদের কাজের একটি অংশ হিসেবে backend pool জুড়ে load balancing করে।

শেষ বিষয়টি গুরুত্বপূর্ণ: **একটি load balancer, যখন এটি আপনার server-এর সামনে application layer-এ কাজ করে, সাধারণত একধরনের নির্দিষ্ট reverse proxy হিসেবেই implement করা হয়।** NGINX এবং HAProxy হলো, technically, reverse proxy যেগুলোতে একটি feature হিসেবে load balancing algorithm অন্তর্ভুক্ত থাকে। তাই সম্পর্কটা "load balancer বনাম reverse proxy" — প্রতিযোগী concept হিসেবে নয় — বরং reverse proxy হলো বিস্তৃত category, এবং load balancing হলো এর সম্পাদিত অনেকগুলো সাধারণ কাজের একটি, যার সাথে থাকে TLS termination, caching, এবং request routing।

### Forward vs Reverse: The Clean Comparison

চলুন একটা পরিষ্কার mental model দিয়ে এই পার্থক্যটা নিশ্চিত করি। একটি forward proxy setup-এ, client proxy সম্পর্কে জানে এবং সেটি configure করে; server-এর কোনো ধারণাই নেই যে কোনো proxy জড়িত আছে — এটি শুধু দেখে request আসছে যা দেখতে একটিমাত্র client (proxy-র IP) থেকে আসা মনে হয়। একটি reverse proxy setup-এ, ঠিক তার উল্টো: server-পক্ষ proxy deploy করে এবং নিয়ন্ত্রণ করে; client-এর কোনো ধারণা নেই যে এর পেছনে একটি সম্পূর্ণ backend fleet রয়েছে — এটি শুধু দেখে একটিমাত্র server-এর মতো কিছু।

অন্যভাবে বললে: forward proxy = client-কে server থেকে লুকায়। reverse proxy = server-কে client থেকে লুকায়। দুটোই anonymity, caching, এবং security সুবিধা প্রদান করে — কিন্তু বিপরীত পক্ষের জন্য।

### Real-World Example

একটি বড় e-commerce company-র কথা ভাবুন। internally, তাদের corporate কর্মীরা একটি forward proxy-র মাধ্যমে internet browse করে যা malicious site filter করে এবং security compliance-এর জন্য traffic log করে — সেই proxy শুধুমাত্র কোম্পানির নিজস্ব client (তার কর্মীদের) রক্ষা ও নিয়ন্ত্রণ করার জন্য বিদ্যমান। সম্পূর্ণ আলাদাভাবে, যখন কোনো customer e-commerce website visit করে, তার request কোম্পানির infrastructure-এর edge-এ থাকা একটি NGINX reverse proxy-তে গিয়ে পৌঁছায়। সেই reverse proxy TLS terminate করে, product page-এর জন্য একটি cache check করে (cache থাকলে সাথে সাথে serve করে), এবং cache না থাকলে, request-কে application server-এর একটি pool জুড়ে load-balance করে। customer-এর browser কখনোই জানতে পারে না যে সেখানে কয়েক ডজন backend server আছে — এটি শুধু reverse proxy-কেই "দেখে"। একই বিস্তৃত concept — traffic relay করা একটি intermediary — কিন্তু সম্পূর্ণ ভিন্ন পক্ষ যাকে সেবা দেওয়া হচ্ছে।

### Recap

একটি proxy হলো client এবং server-এর মাঝে যেকোনো intermediary। একটি forward proxy client-দের সামনে বসে এবং তাদের represent করে — server proxy-কে দেখে, প্রকৃত client-কে নয় — সাধারণত content filtering, anonymity, এবং client-এর হয়ে caching-এর জন্য ব্যবহৃত হয়। একটি reverse proxy server-দের সামনে বসে এবং তাদের represent করে — client proxy-কে দেখে, প্রকৃত backend fleet-কে নয় — সাধারণত TLS termination, caching, internal topology লুকানো, এবং load balancing-এর জন্য ব্যবহৃত হয়। মনে রাখুন: একটি load balancer প্রায়ই কেবল একটি reverse proxy যার মধ্যে load-distribution logic built-in আছে, আলাদা কোনো category নয়।

### What's Next

এখন যেহেতু আমরা reverse proxy-কে একটি server fleet-এর সামনের দরজা হিসেবে বুঝতে পারলাম, পরের ভিডিও এই ধারণাটিকে microservices জগতে আরও এগিয়ে নিয়ে যাবে: API Gateway। আমরা দেখব কীভাবে একটি gateway reverse-proxy concept-এর উপর ভিত্তি করে authentication, rate limiting, এবং কয়েক ডজন backend service জুড়ে unified routing যোগ করে — এবং আমরা Backend-for-Frontend pattern পরিচয় করাব, যখন বিভিন্ন client type-এর ভিন্ন API shape প্রয়োজন হয়।

## Key Takeaways

- একটি proxy হলো এমন যেকোনো intermediary যা কারও হয়ে client এবং server-এর মধ্যে traffic relay করে।
- একটি forward proxy client-কে represent করে — destination server শুধু proxy-কে দেখে, প্রকৃত client-কে নয়। এটি content filtering, anonymity, caching, এবং client traffic monitor করার জন্য ব্যবহৃত হয়।
- একটি reverse proxy server-কে represent করে — client শুধু proxy-কে দেখে, প্রকৃত backend fleet-কে নয়। এটি TLS termination, caching, internal topology লুকানো, এবং load balancing-এর জন্য ব্যবহৃত হয়।
- Load balancing সাধারণত reverse proxy-র শুধু একটি কাজ, আলাদা কোনো architectural category নয়।
- One-line test: forward proxy client-কে server থেকে লুকায়; reverse proxy server-কে client থেকে লুকায়।
