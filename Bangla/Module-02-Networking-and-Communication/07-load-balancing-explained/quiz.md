# Practice & Interview Questions

**১. কেন horizontal scaling-এর জন্য একটা load balancer দরকার হয়?**
Horizontal scaling load handle করার জন্য আরও server যোগ করে, কিন্তু client-দের একটা স্থিতিশীল entry point দরকার এবং traffic-কে fleet জুড়ে বুদ্ধিমত্তার সাথে ছড়িয়ে দেওয়া দরকার। একটা load balancer সেই একক entry point প্রদান করে এবং request বণ্টন করে যাতে কোনো একটা server অতিরিক্ত চাপে না পড়ে অন্যগুলো অলস বসে না থাকে।

**২. Round robin এবং least connections তুলনা করুন। কখন আপনি একটাকে অন্যটার চেয়ে পছন্দ করবেন?**
Round robin বর্তমান load নির্বিশেষে নির্দিষ্ট ক্রমে server-দের মধ্যে ঘোরে — যখন server গুলো identical এবং request-এর cost প্রায় সমান তখন ভালো। Least connections সবচেয়ে কম active connection-ওয়ালা server-এ route করে — যখন request-এর duration/cost উল্লেখযোগ্যভাবে ভিন্ন হয় তখন ভালো, কারণ এটা ইতিমধ্যে ব্যস্ত একটা server-এর উপর আরও কাজ জমা হওয়া এড়িয়ে যায়।

**৩. Layer 4 এবং Layer 7 load balancing-এর মূল পার্থক্য কী?**
L4 transport layer-এ কাজ করে, শুধু IP address এবং port ব্যবহার করে সিদ্ধান্ত নেয়, request content না বুঝেই। L7 application layer-এ কাজ করে, প্রকৃত HTTP request (path, header, cookie) parse করে বুদ্ধিমান, content-aware routing সিদ্ধান্ত নেয়।

**৪. একটা system কেন সীমিত routing বুদ্ধিমত্তা সত্ত্বেও একটা L4 load balancer বেছে নিতে পারে?**
L4 balancer-এর প্রতি-request CPU overhead অনেক কম কারণ তারা application-layer content parse করে না, যা তাদের অত্যন্ত উচ্চ-throughput বা latency-sensitive পরিস্থিতির জন্য, অথবা non-HTTP protocol-এর জন্য যেখানে content-based routing প্রাসঙ্গিক নয়, আদর্শ করে তোলে।

**৫. একটা health check কী, এবং এটা implement করার দুটো সাধারণ পদ্ধতি কী কী?**
Health check হলো একটা mechanism যা load balancer একটা backend server traffic serve করতে সক্ষম কিনা তা নির্ধারণ করতে ব্যবহার করে। Active health check সময়সূচী অনুযায়ী সক্রিয়ভাবে একটা dedicated endpoint (যেমন, `GET /health`) probe করে; passive health check প্রকৃত traffic pattern যেমন error rate বা timeout পর্যবেক্ষণ করে health অনুমান করে।

**৬. Sticky session ব্যাখ্যা করুন এবং IP hashing দিয়ে সেগুলো implement করার একটা অসুবিধা বলুন।**
Sticky session একটা নির্দিষ্ট client-এর request গুলো সামঞ্জস্যপূর্ণভাবে একই backend server-এ route করে, যা প্রায়ই দরকার হয় যখন সেই server in-memory session state ধরে রাখে। IP hashing দিয়ে এটা implement করা ভঙ্গুর কারণ অনেক user একটা একক public IP share করতে পারে (যেমন, NAT/corporate proxy-এর পেছনে), যা অসম load সৃষ্টি করে, এবং একটা server অপসারণ/যোগ করা একসাথে অনেক client-এর জন্য hash mapping পুনর্বিন্যাস করে।

**৭. একটা একক load balancer instance নিজেই একটা single point of failure। আপনি কীভাবে এর চারপাশে design করবেন?**
একাধিক redundant load balancer instance চালান, তাদের জুড়ে DNS round robin-এর মতো একটা mechanism ব্যবহার করে অথবা automatic failover-এর জন্য VRRP/keepalived-এর মতো একটা protocol সহ একটা floating/virtual IP, অথবা একটা cloud provider-এর managed load balancer service-এর উপর নির্ভর করুন, যা একাধিক availability zone জুড়ে সহজাতভাবে redundant।

**৮. দুটো খুব ভিন্ন endpoint type সহ একটা API-এর জন্য load balancing setup design করুন: image upload (বড় payload, ধীর) এবং সহজ lookup (ছোট payload, দ্রুত)। আপনি কীভাবে এই traffic route করবেন?**
একটা L7 load balancer ব্যবহার করুন যা path দিয়ে route করে — যেমন, `/uploads/*` কে বড়, ধীর request-এর জন্য tune/scale করা server-দের একটা pool-এ, এবং `/lookup/*` কে দ্রুত, high-throughput request-এর জন্য optimized একটা আলাদা pool-এ। এটা দুটো traffic type-কে আলাদা করে যাতে ধীর upload-এর একটা burst দ্রুত lookup request-কে অভুক্ত না রাখে, এবং প্রতিটা pool স্বাধীনভাবে scale করা যায়।

**৯. "L4 route করে connection, L7 route করে request" বাস্তবে এর মানে কী?**
একটা L4 balancer প্রতিটা TCP/UDP connection-এ IP/port-এর ভিত্তিতে একটা routing সিদ্ধান্ত নেয় এবং তারপর সেই connection-এর জীবদ্দশায় শুধু packet forward করে। একটা L7 balancer প্রতিটা individual HTTP request-এর জন্য একটা আলাদা routing সিদ্ধান্ত নিতে পারে, এমনকি একই underlying connection-এ multiplexed একাধিক request-এর জন্যও (যেমন, HTTP/2 keep-alive-এর সাথে)।

**১০. প্রতিটা backend server-এর পরিবর্তে load balancer-এ কেন প্রায়ই TLS termination করা হয়?**
L7 load balancer-এ TLS terminate করা certificate management কেন্দ্রীভূত করে, backend server-দের উপর CPU load কমায় (decryption একবারই হয়), এবং operational overhead সহজ করে তোলে — backend server গুলো তখন একটা trusted internal network-এর মধ্যে plain HTTP-এর মাধ্যমে communicate করতে পারে।

**১১. Least connections-এর তুলনায় weighted round robin-এর একটা সম্ভাব্য অসুবিধা কী?**
Weighted round robin অনুমিত server capacity-এর ভিত্তিতে আগে থেকে নির্ধারিত static weight ব্যবহার করে; প্রকৃত load বা request cost পরিবর্তিত হলে এটা real time-এ খাপ খায় না, যেখানে least connections বর্তমান server load-এ সরাসরি প্রতিক্রিয়া দেখায়।

**১২. একটা interview-তে, কীভাবে আপনি একটা video transcoding service-এর জন্য round-robin-এর চেয়ে least-connections বেছে নেওয়া justify করবেন?**
Transcoding job গুলোর duration video-এর length এবং resolution-এর উপর নির্ভর করে ব্যাপকভাবে পরিবর্তিত হয়, তাই round robin একটা server-কে অসমভাবে overload করতে পারে যদি সেটা পরপর কয়েকটা লম্বা job পায়। Least connections সক্রিয়ভাবে হিসাব রাখে প্রতিটা server বর্তমানে কতগুলো job process করছে, উচ্চ-পরিবর্তনশীল-cost কাজের জন্য load আরও ন্যায্যভাবে ছড়িয়ে দেয়।
