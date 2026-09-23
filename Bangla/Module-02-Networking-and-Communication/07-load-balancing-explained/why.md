# Why This Topic Matters: Load Balancing

> **এক বাক্যে:** দশটা server চালানো মূল্যহীন যদি না কোনো কিছু বুদ্ধিমত্তার সাথে সিদ্ধান্ত নেয় প্রতিটা request কোনটাতে যাবে — এবং সেই "কোনো কিছু" এটাও নিশ্চিত করে যে ভাঙা server গুলো থেকে traffic দূরে থাকে।

## The World Before This Idea

আপনি horizontally scale করেছেন: পাঁচটা application server, সবগুলো identical, সবগুলো healthy। এখন কী? User-এর browser শুধু একটা hostname জানে। কাউকে না কাউকে "একটা hostname"-কে "এই পাঁচটা machine-এর একটা"-তে অনুবাদ করতে হবে।

সরল উত্তরটা হলো DNS round-robin: পাঁচটা A record publish করুন এবং client-দের বেছে নিতে দিন। এটা খারাপভাবে কাজ করে। DNS ফলাফল resolver এবং client দ্বারা ব্যাপকভাবে cache করা হয়, তাই বণ্টন অসম হয়। আরও খারাপ, DNS জানে না একটা server বেঁচে আছে কিনা — যখন একটা মারা যায়, আপনার user-দের এক-পঞ্চমাংশ একটা মৃত box-এ পাঠানো হতেই থাকে যতক্ষণ TTL স্থায়ী হয়, এবং real time-এ আপনি এ ব্যাপারে কিছুই করতে পারবেন না।

## The Problems It Solves

### ১. মৃত server যারা traffic পেতেই থাকে
**আপনি যা দেখেন:** একটা backend crash করে। ২০% request ব্যর্থ হতে শুরু করে। কিছুই স্বয়ংক্রিয়ভাবে এটা থামায় না।

**কেন এটা ঘটে:** যা traffic বণ্টন করছে তার backend health সম্পর্কে কোনো ধারণা নেই।

**Load balancing কীভাবে এটা সমাধান করে:** Active health check। Load balancer একটা interval-এ প্রতিটা backend probe করে, এবং ব্যর্থ হওয়া একটা node কয়েক সেকেন্ডের মধ্যে rotation থেকে বাদ দেওয়া হয়। এটা সম্ভবত load balancer-এর *সবচেয়ে* মূল্যবান কাজ — load ছড়িয়ে দেওয়ার চেয়েও বেশি মূল্যবান, কারণ এটা কিছু user-এর জন্য সম্পূর্ণ failure-কে সবার জন্য কোনো failure না হওয়ায় রূপান্তরিত করে।

### ২. অসম load যা fleet নষ্ট করে
**আপনি যা দেখেন:** দুটো server ৯৫% CPU-তে আটকে আছে এবং request জমা হচ্ছে, যখন তিনটে ১৫%-এ বসে আছে। Dashboard-এ গড় utilization ঠিকই দেখায়।

**কেন এটা ঘটে:** সরল বণ্টন উপেক্ষা করে প্রতিটা request আসলে কতটা costly। যদি request-গুলোর কাজের পরিমাণ ব্যাপকভাবে ভিন্ন হয় (একটা search query বনাম একটা health ping), pure round-robin ভারী request গুলো অসমভাবে জমা করবে।

**Load balancing কীভাবে এটা সমাধান করে:** এমন algorithm যা বাস্তবতা বিবেচনায় নেয় — least-connections সেই backend-এ route করে যা বর্তমানে সবচেয়ে কম request handle করছে; weighted variant গুলো ভিন্নধর্মী hardware বিবেচনায় নেয়; least-response-time observed latency হিসাবে নেয়। আপনার request profile-এর জন্য সঠিক algorithm বেছে নেওয়াই raw capacity-কে usable capacity-তে রূপান্তরিত করে।

### ৩. Deploy যার জন্য downtime দরকার
**আপনি যা দেখেন:** প্রতিটা release-এর জন্য একটা maintenance window আছে।

**কেন এটা ঘটে:** যদি traffic সরাসরি server-গুলোতে যায়, তাহলে আপনি সেটার উপর থাকা user-দের বিরক্ত না করে একটা server update করতে পারবেন না।

**Load balancing কীভাবে এটা সমাধান করে:** Connection draining এবং rolling deploy। একটা node-কে rotation থেকে বের করুন, তার in-flight request গুলো শেষ হতে দিন, deploy করুন, health-check করে আবার ফিরিয়ে আনুন, পুনরাবৃত্তি করুন। Load balancer-ই সেই mechanism যা zero-downtime deployment, blue-green, এবং canary release একেবারেই সম্ভব করে তোলে।

### ৪. Routing সিদ্ধান্ত যেগুলোর request-এর ভেতরটা দেখা দরকার
**আপনি যা দেখেন:** আপনি চান `/api/*` API fleet-এ যাক, `/static/*` asset server-এ যাক, এবং ৫% traffic একটা canary build-এ যাক — কিন্তু traffic distributor শুধু IP address এবং port বোঝে।

**কেন এটা ঘটে:** একটা Layer 4 load balancer TCP/UDP connection-এর উপর কাজ করে। এটা দ্রুত এবং protocol-agnostic, কিন্তু এটা path, header, বা cookie পড়তে পারে না।

**এই বিষয়টি কীভাবে এটা সমাধান করে:** L4/L7 পার্থক্য বোঝা। Layer 7 balancer HTTP parse করে, path- এবং header-ভিত্তিক routing, TLS termination, per-request retry, এবং content-aware rule সক্ষম করে — বেশি CPU এবং প্রতি request-এ বেশি latency-এর বিনিময়ে। কোন layer আপনার দরকার তা জানাই আসল design সিদ্ধান্ত।

## The Price You Pay

- **এটা একটা নতুন single point of failure।** যে জিনিসটা আপনাকে server failure থেকে রক্ষা করে সেটা নিজেই fail করতে পারে। বাস্তব deployment-এ failover সহ redundant load balancer দরকার (একটা floating IP, একটা anycast address, বা একটা managed cloud LB)।
- **এটা একটা bottleneck।** সব traffic এর মধ্য দিয়ে প্রবাহিত হয়। এটাকে peak-এর জন্য sized হতে হবে, এবং L7 inspection L4 forwarding-এর চেয়ে উল্লেখযোগ্যভাবে বেশি CPU খরচ করে।
- **Sticky session একটা ফাঁদ।** Session affinity stateful app-এর জন্য প্রলুব্ধকর shortcut, কিন্তু এটা সমান বণ্টনকে দুর্বল করে, একটা node মারা গেলে ভেঙে পড়ে, এবং draining আরও কঠিন করে তোলে। ভালো সমাধান হলো application-কে stateless বানানো।
- **Health check উভয় দিকেই মিথ্যা বলতে পারে।** একটা check যা শুধু নিশ্চিত করে যে process বেঁচে আছে, সেটা এমন একটা node-এ route করতেই থাকবে যার database connection pool শেষ হয়ে গেছে। একটা check যা খুব আক্রমণাত্মক তা একটা transient blip-এর সময় healthy node বাদ দেবে এবং একটা incident-কে বাড়িয়ে তুলবে।

## When You Need It — and When You Don't

| আপনার একটা দরকার যখন | আপনি এটা এড়িয়ে যেতে পারেন যখন |
|---|---|
| আপনি যেকোনো কিছুর একাধিক instance চালান | সত্যিকারের single-instance, low-stakes system |
| আপনি zero-downtime deploy চান | Downtime window গ্রহণযোগ্য |
| আপনার এক জায়গায় TLS termination দরকার | — |
| আপনি canary বা blue-green release চান | — |
| আপনার path, header, বা geography দিয়ে traffic shape করা দরকার | — |

## Why This Shows Up in Interviews

একটা load balancer মূলত প্রতিটা system design diagram-এ দেখা যায়, যার মানে একটা আঁকা আপনাকে কিছুই এনে দেয় না। যা credit এনে দেয় তা হলো follow-up: কোন algorithm এবং কেন, L4 নাকি L7 এবং কেন, health check কীভাবে কাজ করে, load balancer নিজে fail করলে কী হয়, এবং কীভাবে আপনি sticky session এড়াবেন। Interviewer-রা প্রায়ই শেষটা নির্দিষ্টভাবে পরীক্ষা করে, কারণ এটা এমন মানুষদের আলাদা করে যারা একটা fleet পরিচালনা করেছে তাদের থেকে যারা শুধু এটা সম্পর্কে পড়েছে।

## How It Connects

Load balancing হলো **horizontal scaling**-এর ব্যবহারিক সক্ষমকারী এবং **fault tolerance**-এর একটা মূল mechanism। এটা **reverse proxy**-এর সাথে ঘনিষ্ঠভাবে সম্পর্কিত (একটা load balancer এর একটা বিশেষায়িত রূপ), **API gateway**-এর পাশে বসে থাকে (যা এর উপরে auth, rate limiting, এবং aggregation যোগ করে), এবং **consistent hashing** ব্যবহার করে যখন backend-গুলো state ধরে রাখে এবং আপনার একটা স্থিতিশীল key-to-node mapping দরকার। **Zero-downtime deployment** এর draining এবং health-check আচরণের উপর সরাসরি নির্মিত।

**পরবর্তী:** [Forward Proxy vs Reverse Proxy](../08-forward-proxy-vs-reverse-proxy/why.md) — intermediary-দের সেই বৃহত্তর পরিবার যার অংশ load balancer।
