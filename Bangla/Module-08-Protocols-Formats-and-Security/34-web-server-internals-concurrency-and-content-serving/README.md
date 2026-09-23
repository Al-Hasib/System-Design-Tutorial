# Web Server Internals: Concurrency, Threading & Content Serving

**কঠিনতার মাত্রা:** Intermediate

## শেখার লক্ষ্যসমূহ (Learning Objectives)

- একটি web server আসলে একটি incoming connection, request, এবং response নিয়ে কী করে তা ব্যাখ্যা করা।
- তিনটি প্রধান concurrency model তুলনা করা: process-per-request, thread-per-request, এবং event loop।
- C10K problem কী এবং কেন এটি one-thread-per-connection ডিজাইন থেকে সরে আসতে বাধ্য করেছিল তা বোঝা।
- static content serving কে dynamic content serving থেকে আলাদা করা এবং প্রতিটি কেন ভিন্নভাবে হ্যান্ডেল করা হয় তা ব্যাখ্যা করা।
- connection keep-alive, thread/worker pool, এবং reverse-proxy caching কীভাবে একসাথে কাজ করে বাস্তব-জগতের throughput নির্ধারণ করতে তা ব্যাখ্যা করা।

## স্ক্রিপ্ট (Script)

### Hook / ভূমিকা

আমরা এতক্ষণ load balancer নিয়ে কথা বলেছি, যা ট্রাফিককে একাধিক server-এ ছড়িয়ে দেয়, আর reverse proxy নিয়েও, যা সেগুলোর সামনে বসে থাকে — কিন্তু যখন দশ হাজার request একসাথে একটি server-এ এসে পড়ে, তখন সেই server-এর *ভেতরে* আসলে কী ঘটে? এই অংশটা এমন যে, বেশিরভাগ backend engineer বছরের পর বছর একটি framework ব্যবহার করেও এ নিয়ে কখনো ভাবার প্রয়োজন বোধ করেন না — যতক্ষণ না একদিন তাদের app load-এর নিচে ভেঙে পড়ে এবং postmortem-এ লেখা থাকে "thread pool exhaustion" অথবা "the event loop got blocked।" আজ আমরা web server-টিকেই খুলে দেখব: এটি কীভাবে concurrency হ্যান্ডেল করে, কেন কিছু framework একটি মাত্র box-এ 50,000 connection সামলায় আর অন্যগুলো 500-তেই কাত হয়ে যায়, এবং কেন একটি static image serve করা আর একটি database-backed API response serve করা মৌলিকভাবে ভিন্ন কাজ।

### একটি Web Server আসলে কী করে

Framework-নির্দিষ্ট জাদু বাদ দিলে, একটি web server প্রতিটি request-এর জন্য একটি loop-এ চারটি কাজ করে: একটি connection accept করা, request পড়া এবং parse করা, response তৈরির জন্য যা প্রয়োজন তা করা (যা হয়তো শুধু একটি file পড়া, অথবা একটি database ও তিনটি downstream service-কে ছুঁয়ে আসাও হতে পারে), এবং response ফেরত লেখা। "web server performance" নিয়ে সমস্ত প্রশ্ন আসলে এই তৃতীয় ধাপ — প্রকৃত কাজ — একসাথে অনেক request-এর জন্য একই *সময়ে* কতটা দক্ষভাবে করা যায় তার উপর নির্ভর করে, যেন একটি ধীর request বাকি সবগুলোকে আটকে না দেয়।

### Concurrency Models

ঐতিহাসিকভাবে একই সাথে অনেক request হ্যান্ডেল করার জন্য তিনটি বড় পদ্ধতি ছিল।

**Process-per-request** — মূল CGI model — প্রতিটি আগত request-এর জন্য একটি সম্পূর্ণ নতুন operating system process fork করে। এটি সহজ এবং অত্যন্ত isolated (একটি crashing request অন্যটিকে নামিয়ে দিতে পারে না), কিন্তু process গুলো ভারী: একটি তৈরি করা, তাকে নিজস্ব memory space দেওয়া, এবং আবার ভেঙে ফেলা ধীর এবং একটি মেশিনে কয়েকশোর বেশি concurrent request-এর বেশি scale করতে পারে না।

**Thread-per-request** হলো ক্লাসিক model, যা Apache-এর `mpm_prefork`/`worker` এবং অনেক ঐতিহ্যবাহী Java application server ব্যবহার করে: প্রতিটি আগত connection নিজস্ব OS বা lightweight thread পায়, যা response পাঠানোর মতো কিছু না থাকা পর্যন্ত I/O-এর (যেমন একটি database query-র জন্য অপেক্ষা করা) উপর block করে থাকে। Thread গুলো process-এর চেয়ে সস্তা, কিন্তু প্রতিটির জন্যই বাস্তব memory খরচ হয় (প্রায়ই এক megabyte বা তার বেশি stack space) এবং OS-কে সেগুলোর মধ্যে context-switch করতে হয়, যা হাজার হাজার thread থাকলে বাস্তব overhead তৈরি করে। এখান থেকেই বিখ্যাত **C10K problem**-এর জন্ম — একটি 1999 সালের প্রবন্ধ, যেখানে দেখানো হয়েছিল যে one-thread-per-connection architecture দিয়ে দশ হাজার concurrent connection হ্যান্ডেল করা একটি দেয়ালে ধাক্কা খাচ্ছিল, কারণ OS এত বেশি thread সস্তায় schedule করতে পারত না।

**Event loop / asynchronous, non-blocking I/O** হলো আধুনিক সমাধান, যা Node.js, Nginx, এবং Python-এর (asyncio), Java-র (Netty) মতো async framework-এ ব্যবহৃত হয়। প্রতি connection-এ একটি thread-এর বদলে, একটি ছোট, নির্দিষ্ট সংখ্যক thread-এর pool (কখনো কখনো শুধুই একটি) কখনো block না করে অনেক connection হ্যান্ডেল করে: যখন একটি request-কে I/O-এর জন্য অপেক্ষা করতে হয় — একটি database call, একটি file read, অন্য একটি service-এ call — তখন event loop একটি callback register করে অন্য connection সার্ভ করতে চলে যায়, এবং শুধুমাত্র সেই I/O সম্পন্ন হলেই আবার এই connection-এ ফিরে আসে। এর মানে হলো হাজার হাজার "অপেক্ষমাণ" connection প্রায় কিছুই খরচ করে না, কারণ অপেক্ষার সময় এদের কেউই একটি dedicated thread আটকে রাখছে না। Trade-off হলো: একটি একক দীর্ঘস্থায়ী, CPU-bound কাজ (যেমন একটি ভারী synchronous computation) যতগুলো connection সার্ভ করছে তার *পুরো* event loop-কেই block করে দেয় — এটাই সেই "আমি ভুলবশত event loop block করে ফেলেছি" bug, যা load-এর নিচে একটি Node.js server-কে নামিয়ে দেয়। বাস্তবে, আজকের বেশিরভাগ production system-ই hybrid: I/O-bound কাজ হ্যান্ডেল করে একটি async event loop, আর সত্যিকারের CPU-bound যেকোনো কিছুর জন্য থাকে একটি worker/thread pool।

### Static বনাম Dynamic Content

সব request সমান কাজ নয়। **Static content** — একটি image, একটি CSS file, একটি pre-built JavaScript bundle — প্রতি request-এ কখনো পরিবর্তিত হয় না; server-এর কাজ শুধু "disk (বা memory) থেকে এই bytes গুলো পড়ে পাঠিয়ে দাও," যে কারণে static asset গুলো ব্যাপকভাবে cache করা হয় (Module 4-এর CDN মনে করুন) এবং কেন dedicated static file server (বা একটি CDN edge) সেগুলোকে আপনার application code-এর মধ্য দিয়ে route করার চেয়ে অনেক দ্রুত এবং সস্তায় serve করতে পারে। **Dynamic content**-এর জন্য server-কে এই নির্দিষ্ট request-এর জন্য বাস্তব কাজ করতে হয় — একটি database query করা, business logic প্রয়োগ করা, একটি response personalize করা — এরপর তার কাছে ফেরত পাঠানোর মতো কিছু থাকে। এই কারণেই production architecture গুলো application server-এর সামনে একটি reverse proxy বা CDN বসায়: static asset এবং cacheable response গুলো একদম edge-এই serve (বা cache) হয়ে যায়, এবং শুধুমাত্র যেসব request-এর সত্যিই dynamic computation দরকার সেগুলোই আপনার application-এর concurrency model পর্যন্ত পৌঁছায়। আপনার app server-এ যত কম request পৌঁছাবে, আপনার concurrency model-এর সীমা তত বেশি প্রসারিত হবে।

### বাস্তব-জগতের উদাহরণ

একটি e-commerce product page-এর কথা ভাবুন। Product image এবং page-এর CSS/JS bundle static — সরাসরি একটি CDN edge cache থেকে serve হয়, আপনার application server-কে কখনো স্পর্শ করে না। বর্তমান price এবং stock ফেরত দেওয়া `GET /products/123` API call টি dynamic — এটিকে একটি database (অথবা Module 4-এর cache-aside layer) স্পর্শ করতেই হবে এবং সামান্য বাসি price নিয়ে স্বাচ্ছন্দ্যবোধ না করলে একটি CDN edge থেকে serve করা যাবে না। এই API যদি Node.js-এর event loop-এর উপর তৈরি হয়, তাহলে দশ হাজার concurrent `GET /products/123` call database-এর জন্য অপেক্ষা করার সময় "in flight" ধরে রাখা সস্তা — কোনোটির জন্যই event loop block হয়ে থাকে না। কিন্তু যদি কোনো request-কে synchronously একটি PDF invoice তৈরি করতে হয় (একটি সত্যিকারের CPU-heavy কাজ), তাহলে সেটা সরাসরি event loop thread-এ করলে সেটি শেষ না হওয়া পর্যন্ত server-এর অন্য প্রতিটি request আটকে যাবে — এটাই ঠিক সেই ধরনের কাজ, যা আপনি এর পরিবর্তে একটি background worker pool বা আলাদা service-এ পাঠাবেন।

### Recap

একটি web server-এর মূল loop হলো accept, read, process, write — এবং সমস্ত performance-এর গল্পটা হলো এটি কীভাবে "process" ধাপটি concurrently হ্যান্ডেল করে। Process-per-request এবং thread-per-request সহজ, কিন্তু OS-স্তরের overhead-এর কারণে কয়েক হাজার connection-এর বেশি scale করে না — এটাই C10K problem। Event-loop, non-blocking I/O শুধু অপেক্ষারত একটি connection-এর জন্য কখনো একটি thread উৎসর্গ না করে দশ হাজারেরও বেশি connection পর্যন্ত scale করতে পারে, কিন্তু একটিমাত্র CPU-bound কাজ বাকি সবাইকে block করে দেওয়ার ঝুঁকিতে থাকে। আর static content-এর কখনোই এই concurrency model পর্যন্ত পৌঁছানো উচিত নয় — এটি একটি CDN বা reverse-proxy edge-এ থাকার কথা, যাতে আপনার application server-এর সীমিত concurrency শুধু সেই dynamic কাজের জন্য থাকে যার সত্যিই তা প্রয়োজন।

### এরপর কী

আমরা এখন কভার করেছি বাইট কীভাবে চলাচল করে (TCP/UDP/gRPC) এবং একটি server কীভাবে একইসাথে অনেক request হ্যান্ডেল করে। পরবর্তী প্রশ্ন হলো: দুটি service যখন data আদান-প্রদান করার সিদ্ধান্ত নেয়, সেই data আসলে কোন format-এ থাকে? পরের video-তে আমরা JSON, XML, এবং Protocol Buffers তুলনা করব সেইসব বিষয়ে যা সত্যিই গুরুত্বপূর্ণ — bandwidth, parsing speed, এবং সময়ের সাথে schema পরিবর্তন হলে প্রতিটি কতটা মসৃণভাবে তা সামলায়।

## মূল বিষয়সমূহ (Key Takeaways)

- প্রতিটি web server প্রতি request-এ একই চারটি ধাপ করে — accept, read, process, write — এবং performance সম্পূর্ণভাবে নির্ভর করে "process" কীভাবে concurrently scale করে তার উপর।
- Process-per-request এবং thread-per-request model গুলো OS memory এবং context-switching overhead-এর কারণে কয়েক হাজার connection-এর বেশি scale করে না — এটাই C10K problem।
- Event-loop/async, non-blocking I/O (Nginx, Node.js, asyncio) I/O wait-এ কখনো একটি thread আটকে না রেখে দশ হাজারেরও বেশি connection পর্যন্ত scale করে, কিন্তু loop-এ একটি blocking CPU-bound task সেটি সার্ভ করা প্রতিটি connection-কে আটকে দেয়।
- Static content (image, CSS, prebuilt bundle) একটি CDN/reverse-proxy edge থেকে serve করা উচিত, কখনোই application concurrency logic-এর মধ্য দিয়ে route করা উচিত নয়।
- বেশিরভাগ production system hybrid: I/O-bound কাজের জন্য একটি async event loop, আর সত্যিকারের CPU-bound task-এর জন্য একটি worker/thread pool।
