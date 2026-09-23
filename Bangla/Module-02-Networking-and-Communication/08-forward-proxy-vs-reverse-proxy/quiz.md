# Practice & Interview Questions

**১. এক বাক্যে, forward proxy এবং reverse proxy-র মধ্যে পার্থক্য কী?**
একটি forward proxy client-দের সামনে বসে এবং তাদের server থেকে লুকায়/represent করে; একটি reverse proxy server-দের সামনে বসে এবং তাদের client থেকে লুকায়/represent করে।

**২. একটি forward proxy setup-এ destination server কার identity দেখে?**
Destination server শুধু forward proxy-র IP/identity দেখে — প্রকৃত originating client সম্পর্কে এর কোনো visibility নেই।

**৩. একটি forward proxy-র জন্য দুটি বাস্তব-জগতের use case দিন।**
Corporate content filtering/কর্মীদের outbound traffic monitor করা, এবং client-এর identity বা location anonymize/mask করা (যেমন, geo-restriction এড়ানো বা internal client IP লুকানো)।

**৪. একটি reverse proxy-র জন্য দুটি বাস্তব-জগতের use case দিন।**
Backend server-এর হয়ে TLS/SSL termination, এবং backend application server-এর একটি pool জুড়ে request caching বা load balancing করা।

**৫. একটি load balancer কি reverse proxy থেকে একটি আলাদা concept, নাকি এর একটি বিশেষ ক্ষেত্র?**
একটি L7 load balancer সাধারণত reverse proxy-র একটি বিশেষ ক্ষেত্র — এমন একটি reverse proxy যার একটি feature হলো load-distribution algorithm। Reverse proxy হলো বিস্তৃত category; load balancing হলো TLS termination এবং caching-এর মতো অন্যান্য কাজের মধ্যে একটি সাধারণ কাজ।

**৬. একটি reverse proxy কীভাবে backend server-এর জন্য security উন্নত করে?**
এটি external client থেকে backend server-এর internal network topology ও প্রকৃত IP address লুকিয়ে রাখে, তাই attacker-রা সরাসরি সেগুলোকে target করতে পারে না; এটি security control (TLS, WAF rule, rate limiting) প্রয়োগের জায়গাকেও কেন্দ্রীভূত করে।

**৭. একটি কোম্পানি কর্মঘণ্টায় কর্মীদের নির্দিষ্ট website-এ প্রবেশ ঠেকাতে চায় এবং compliance-এর জন্য সব outbound traffic log করতে চায়। তাদের কোন ধরনের proxy দরকার, এবং কেন?**
একটি forward proxy — এটি client (কর্মী)-পক্ষে deploy করা হয়, network ছেড়ে যাওয়ার আগে outbound request inspect/filter/block করতে পারে, এবং সব traffic log করতে পারে কারণ প্রতিটি কর্মীর request এর মধ্য দিয়ে যায়।

**৮. একটি reverse proxy setup-এ client-এর সাধারণত কোনো ধারণা থাকে না যে কতগুলো backend server বিদ্যমান, কেন?**
কারণ reverse proxy client-এর কাছে একটি একক unified address/identity উপস্থাপন করে এবং internally request-কে যে backend server-এ পাঠাতে চায় সেখানে forward করে — backend topology client-এর দৃষ্টিকোণ থেকে সম্পূর্ণভাবে abstract করা থাকে।

**৯. একটি একক infrastructure কি একসাথে forward proxy এবং reverse proxy দুটোই ব্যবহার করতে পারে? একটি উদাহরণ দিন।**
হ্যাঁ — একটি কোম্পানি তার কর্মীদের outbound internet traffic-এর জন্য একটি forward proxy চালাতে পারে (client-দের রক্ষা/monitor করার জন্য) এবং একই সাথে আলাদাভাবে তার public-facing web server-এর সামনে একটি reverse proxy (যেমন NGINX) চালাতে পারে (server-পক্ষকে রক্ষা/optimize করার জন্য)। এগুলো সম্পূর্ণ ভিন্ন traffic flow এবং purpose-এর জন্য কাজ করে।

**১০. একটি reverse proxy কেন response cache করতে পারে, এবং কোন ধরনের content এর জন্য সবচেয়ে উপযুক্ত?**
Reverse proxy-তে caching করলে একই request-এর জন্য বারবার backend server-এ যাওয়া এড়ানো যায়, যা load এবং latency কমায়। এটি static বা কদাচিৎ পরিবর্তনশীল content (image, CSS/JS, এমন product page যা per-request পরিবর্তিত হয় না)-এর জন্য সবচেয়ে উপযুক্ত, অত্যন্ত personalized বা দ্রুত পরিবর্তনশীল data-র জন্য নয়।

**১১. সব outbound traffic একটি একক forward proxy-র মধ্য দিয়ে route করার সম্ভাব্য অসুবিধা কী?**
এটি সব outbound client traffic-এর জন্য একটি single point of failure এবং সম্ভাব্য bottleneck হয়ে ওঠে — যদি এটি down হয়ে যায় বা overload হয়, তাহলে এর পেছনের কোনো client internet-এ পৌঁছাতে পারে না; এটিকেও যেকোনো critical infrastructure component-এর মতো scale ও redundant করতে হয়।

**১২. একজন interviewer-কে আপনি কীভাবে বোঝাবেন যে "reverse proxy" এবং (পরের ভিডিওতে আলোচিত) "API Gateway" সম্পর্কিত কিন্তু অভিন্ন নয়?**
একটি API Gateway reverse-proxy concept-এর উপর ভিত্তি করে তৈরি — এটিও backend service-এর সামনে বসে এবং তাদের topology লুকায় — কিন্তু এটি API ও microservices-এর জন্য নির্দিষ্ট application-aware capability যোগ করে, যেমন authentication/authorization, rate limiting, request/response transformation, এবং একাধিক service-এ call aggregate করা, যা একটি plain reverse proxy সাধারণত যা করে তার চেয়ে বেশি।
