# Load Balancing Explained (Algorithms & L4 vs L7)

**Difficulty:** Intermediate

## Learning Objectives

- ব্যাখ্যা করুন কেন সিস্টেম horizontally scale করার পর load balancer অপরিহার্য হয়ে ওঠে।
- সাধারণ load balancing algorithm গুলো তুলনা করুন: round robin, least connections, weighted variants, IP hash, এবং consistent hashing।
- Layer 4 (transport) এবং Layer 7 (application) load balancing-এর মধ্যে পার্থক্য করুন এবং কখন কোনটি ব্যবহার করবেন তা জানুন।
- বর্ণনা করুন কীভাবে health check একটি load balancer-এর server pool কে সঠিক রাখে।
- একটি load balancing setup-এ single point of failure চিহ্নিত করুন এবং কীভাবে তা দূর করা যায়।

## Script

### Hook / Intro

আগের ভিডিওতে আমরা প্রতিষ্ঠা করেছিলাম যে HTTP stateless — যা দারুণ ব্যাপার, কারণ এর মানে হলো যেকোনো server টেকনিক্যালি যেকোনো request-এর উত্তর দিতে পারে। কিন্তু এতে একটা স্বাভাবিক প্রশ্ন আসে: যদি আমার কাছে দশটা identical server থাকে যারা একটা request handle করতে সক্ষম, তাহলে আসলে কোনটা request পাবে? এই সিদ্ধান্ত — যা প্রতিটি request-এ নেওয়া হয়, scale-এ প্রতি সেকেন্ডে হাজার হাজার বা লক্ষ লক্ষ বার — এটাই একটা load balancer-এর কাজ। আজ আমরা ভেঙে দেখব load balancer গুলো ঠিক কীভাবে সিদ্ধান্ত নেয়, তারা যেসব প্রধান algorithm ব্যবহার করে, এবং Layer 4 বনাম Layer 7-এ balancing করার মৌলিক পার্থক্য। এটি system design interview-তে সবচেয়ে নিয়মিতভাবে জিজ্ঞাসিত বিষয়গুলোর একটি, তাই চলুন এটাকে একদম পোক্ত করে ফেলি।

### Why We Need Load Balancing

কল্পনা করুন একটি একক server আপনার সব traffic handle করছে। এর একটা hardware ceiling আছে — প্রতি সেকেন্ডে সর্বোচ্চ কতগুলো request handle করা যাবে তার একটা সীমা, তারপর CPU, memory, বা network bandwidth bottleneck হয়ে দাঁড়ায়। Vertical scaling — আরও বড় machine কেনা — কিছুদিন সাহায্য করে, কিন্তু এটা ব্যয়বহুল এবং এর একটা কঠিন সীমা আছে। তাই আমরা horizontally scale করি: একই application অনেকগুলো server-এ চালাই। কিন্তু এখন client-দের কথা বলার জন্য একটা একক, স্থিতিশীল address দরকার, এবং request গুলোকে বুদ্ধিমত্তার সাথে fleet জুড়ে ছড়িয়ে দিতে হবে। এটাই load balancer: এটা আপনার server pool-এর সামনে বসে থাকে, সমস্ত আগত traffic receive করে, এবং সিদ্ধান্ত নেয় কোন backend server প্রতিটি request handle করবে।

শুধু load ছড়িয়ে দেওয়ার বাইরেও, একটি ভালো load balancer আপনাকে আরও দুটি বিশাল সুবিধা দেয়। প্রথমত, availability: যদি একটি server crash করে, load balancer health check-এর মাধ্যমে তা শনাক্ত করে এবং সহজেই সেটিতে traffic পাঠানো বন্ধ করে দেয় — user-দের কাছে failure টা অদৃশ্য হয়ে যায়। দ্বিতীয়ত, এটি zero-downtime deployment সম্ভব করে: আপনি server-গুলোকে rotation থেকে বের করতে, update করতে, এবং আবার যোগ করতে পারবেন, user-রা কিছু টের না পেয়েই।

### Load Balancing Algorithms

চলুন দেখি একটি load balancer আসলে কীভাবে একটা server বেছে নেয়। এটা "শুধু randomly একটা বেছে নেওয়া"-র চেয়ে অনেক বেশি সূক্ষ্ম।

**Round robin** সবচেয়ে সহজ: request গুলো strict rotation-এ server-দের কাছে হস্তান্তর করা হয় — server 1, তারপর 2, তারপর 3, তারপর আবার 1। যখন সব server identical এবং সব request-এর cost প্রায় সমান, তখন এটা সহজ এবং ন্যায্য। **Weighted round robin** এটাকে আরও প্রসারিত করে, বেশি শক্তিশালী server-কে বেশি weight দিয়ে, যাতে একটা শক্তিশালী server অনুপাতে বেশি request পায়।

**Least connections** প্রতিটি নতুন request-কে সেই server-এ পাঠায় যার বর্তমানে সবচেয়ে কম active connection আছে। এটা round robin-এর চেয়ে বেশি বুদ্ধিমান যখন request-এর cost ব্যাপকভাবে ভিন্ন হয় — একটি ধীর request প্রসেস করতে থাকা server-এর উপর আর চাপ জমা হবে না। **Weighted least connections** আবার server capacity-কে হিসাবে নেয়।

**IP hash** client-এর IP address-এর একটি hash গণনা করে এবং সেই client-কে একই backend server-এ সামঞ্জস্যপূর্ণভাবে map করতে এটি ব্যবহার করে। এটা তখন কাজে লাগে যখন আপনার "sticky session" দরকার — উদাহরণস্বরূপ, যদি একটি server সেই user-এর জন্য কিছু in-memory session state ধরে রাখে। ট্রেড-অফ হলো, যদি একটি server down হয়ে যায়, তাহলে যারা সেটাতে hash হয়েছিল তাদের সবাইকে পুনর্বণ্টন করতে হবে, এবং client IP-এর বণ্টন skewed হলে hash-based পদ্ধতি অসম load সৃষ্টি করতে পারে।

এছাড়াও আছে **least response time**, যা active connection এবং observed latency উভয়কেই হিসাবে নেয়, এবং — উল্লেখ করার মতো যেহেতু আমরা কোর্সে পরে এটা বিস্তারিতভাবে কভার করব — **consistent hashing**, যা server যোগ বা বাদ দেওয়ার সময় পুনর্বণ্টন সমস্যাকে কমিয়ে আনে, সাধারণত plain web traffic-এর চেয়ে cache এবং shard routing-এর জন্য ব্যবহৃত হয়।

### Layer 4 vs Layer 7 Load Balancing

এটাই সেই পার্থক্য যা interviewer-রা যাচাই করতে ভালোবাসে, তাই চলুন সুনির্দিষ্ট হই। এটা নির্দেশ করে load balancer OSI networking model-এর কোন layer-এ কাজ করে।

একটি **Layer 4 (L4) load balancer** transport layer-এ কাজ করে — এটা TCP/UDP তথ্য দেখে: source এবং destination IP address এবং port। এটা packet-এর প্রকৃত content-এর ভিতরে না তাকিয়ে বা HTTP একেবারেই না বুঝে routing সিদ্ধান্ত নেয়। এটা শুধু দেখে "IP A থেকে IP B-তে connection" এবং packet forward করে। এটা L4 balancing-কে অত্যন্ত দ্রুত এবং low-overhead করে তোলে — এটা ন্যূনতম CPU cost-এ বিশাল throughput push করতে পারে — কিন্তু এটা এই অর্থে "dumb"-ও যে এটা request-এর content, যেমন URL path বা header-এর ভিত্তিতে সিদ্ধান্ত নিতে পারে না।

একটি **Layer 7 (L7) load balancer** application layer-এ কাজ করে — এটা আসলেই HTTP বোঝে। এটা URL path, header, cookie, এমনকি request body পড়তে পারে এবং সেই অনুযায়ী route করতে পারে। এর মানে একটি L7 balancer `/api/images/*` কে একটি service-এ এবং `/api/checkout/*` কে অন্য একটিতে পাঠাতে পারে, backend server-দের পক্ষে TLS terminate করতে পারে, header inject বা rewrite করতে পারে, এবং শুধু IP hashing-এর পরিবর্তে cookie-এর মাধ্যমে sticky session implement করতে পারে। এর cost হলো প্রতি request-এ বেশি CPU overhead, কারণ এটাকে প্রতিটি HTTP request আসলেই parse এবং বুঝতে হয়, শুধু packet এদিক-ওদিক না করে।

মনে রাখার একটা সহজ উপায়: L4 route করে connection, L7 route করে request। বাস্তবে, বেশিরভাগ আধুনিক production system L7 load balancer ব্যবহার করে — যেমন NGINX, HAProxy L7 mode-এ, AWS Application Load Balancer, বা Envoy — কারণ বুদ্ধিমান routing capability-গুলো অতিরিক্ত CPU cost-এর যোগ্য, এবং hardware সেই cost-কে বেশিরভাগ scale-এ যথেষ্ট নগণ্য করে দিয়েছে। L4 balancer — যেমন AWS Network Load Balancer — এখনও অত্যন্ত উচ্চ-throughput, low-latency পরিস্থিতিতে, বা সম্পূর্ণরূপে non-HTTP protocol-এর জন্য দুর্দান্ত।

### Health Checks and Avoiding a Single Point of Failure

একটি load balancer ততটাই ভালো যতটা এটা জানে কোন server গুলো আসলে healthy। এটা করা হয় health check-এর মাধ্যমে — load balancer পর্যায়ক্রমে প্রতিটি backend-কে ping করে, প্রায়ই একটি হালকা `/health` HTTP endpoint দিয়ে, এবং যদি একটি server নির্দিষ্ট সংখ্যকবার সঠিকভাবে সাড়া দিতে ব্যর্থ হয়, তাহলে এটা স্বয়ংক্রিয়ভাবে rotation থেকে বাদ পড়ে। এটাই সেই জিনিস যা end user-এর কাছে failure-কে অদৃশ্য করে তোলে।

কিন্তু এখানে একটা সূক্ষ্ম বিষয় আছে যা interview-তে অনেককে বিভ্রান্ত করে: load balancer নিজেই একটা server, এবং একটি একক load balancer একটা single point of failure। যদি এটা down হয়ে যায়, আপনার সম্পূর্ণ healthy backend server-এর fleet unreachable হয়ে যায়। Production system এটা সমাধান করে redundancy দিয়ে — DNS round robin-এর মতো একটা mechanism-এর পেছনে একাধিক load balancer instance চালিয়ে, VRRP-এর মতো একটা protocol সহ একটা floating/virtual IP, বা একটা cloud provider-এর managed load balancer service যা availability zone জুড়ে সহজাতভাবে redundant। সাধারণ নীতি: load balancing নিজেই কখনো সেই bottleneck বা single point of failure হতে দেবেন না যা এটি দূর করার জন্য বানানো হয়েছিল।

### Real-World Example

একটা video streaming platform-এর API tier-এর কথা ভাবুন একটা বড় live event চলাকালীন। লক্ষ লক্ষ client একসাথে connect করে। একটি L7 load balancer edge-এ বসে থাকে, TLS terminate করে যাতে backend server-দের প্রত্যেককে আলাদাভাবে certificate manage করতে না হয়, এবং এটা path অনুযায়ী route করে: `/video/manifest`-এর request গুলো একটা হালকা-loaded manifest service-এ যায়, `/chat/messages` একটা আলাদা chat service-এ যায়, এবং health check ক্রমাগত উভয় pool monitor করে। Video manifest pool-এর ভেতরে, এটা least-connections ব্যবহার করতে পারে, কারণ manifest request-এর cost video quality tier-এর উপর নির্ভর করে পরিবর্তিত হতে পারে। যদি একটা manifest server load-এর নিচে error দিতে শুরু করে, health check কয়েক সেকেন্ডের মধ্যে তা ধরে ফেলে এবং traffic স্বয়ংক্রিয়ভাবে reroute হয় — user-রা শুধু failed request-এর বদলে সামান্য বেশি, কিন্তু non-fatal, গড় latency দেখে।

### Recap

Load balancer গুলো বিদ্যমান কারণ horizontal scaling তখনই কাজ করে যখন traffic বুদ্ধিমত্তার সাথে অনেকগুলো identical server জুড়ে ছড়িয়ে দেওয়া হয়। Algorithm — round robin, least connections, IP hash, এবং তাদের weighted variant-গুলো — নির্ধারণ করে কীভাবে সেই ছড়িয়ে দেওয়া হয়। L4 বনাম L7 পার্থক্যটা হলো load balancer সিদ্ধান্ত নিতে কোন তথ্য ব্যবহার করে সেই সম্পর্কে: L4 শুধু IP/port দেখে এবং অত্যন্ত দ্রুত; L7 নিজে HTTP বোঝে এবং সামান্য বেশি CPU cost-এ অনেক বুদ্ধিমান routing সিদ্ধান্ত নিতে পারে। Health check pool-কে সঠিক রাখে, এবং redundant load balancer গুলো balancer নিজেকে single point of failure হওয়া থেকে প্রতিরোধ করে।

### What's Next

Load balancer একটা বিস্তৃত ধারণার একটা রূপ: client এবং প্রকৃত server-দের মাঝে বসে থাকা কিছু একটা। পরের ভিডিওতে, আমরা zoom out করব এবং সাধারণভাবে proxy কভার করব — বিশেষত forward proxy-র মধ্যে পার্থক্য, যা client-দের প্রতিনিধিত্ব ও সুরক্ষা করে, এবং reverse proxy, যা server-দের প্রতিনিধিত্ব ও সুরক্ষা করে। আপনি দেখবেন কীভাবে একটা reverse proxy এবং একটা load balancer বাস্তবে প্রায়ই overlap করে।

## Key Takeaways

- Load balancer গুলো server pool জুড়ে traffic বণ্টন করে, যা horizontal scaling, high availability, এবং zero-downtime deployment সম্ভব করে।
- সাধারণ algorithm গুলো: round robin, weighted round robin, least connections, IP hash — প্রতিটি ভিন্ন ভিন্ন traffic pattern-এর জন্য উপযুক্ত।
- L4 load balancing IP/port (transport layer)-এর ভিত্তিতে route করে — দ্রুত কিন্তু content-blind। L7 load balancing HTTP বোঝে এবং path, header, cookie-এর ভিত্তিতে route করে — বুদ্ধিমান কিন্তু বেশি ব্যয়বহুল।
- Health check একটা load balancer-কে rotation থেকে স্বয়ংক্রিয়ভাবে unhealthy server সরাতে দেয়, যা user-দের কাছে failure অদৃশ্য করে তোলে।
- একটি একক load balancer একটা single point of failure — production system গুলো এটা এড়াতে redundant load balancer চালায়।
