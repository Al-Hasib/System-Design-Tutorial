# কেন এই Topic গুরুত্বপূর্ণ: Microservices Communication ও Service Discovery

> **এক বাক্যে:** একটি system-কে service-এ ভাগ করলে সাথে সাথে দুটি সমস্যা তৈরি হয় যা আগে ছিল না — একটি service কীভাবে অন্য একটি service খুঁজে পাবে যার instance ক্রমাগত বদলে যাচ্ছে, আর সেই অন্য service ধীর বা নষ্ট হয়ে গেলে caller-এর কী হবে।

## এই ধারণার আগের জগৎ

আপনি service-এ ভাগ করেছেন। এখন order service-কে inventory service কল করতে হবে। সেটা কোথায় আছে?

স্বজ্ঞাত (instinctive) উত্তর হলো একটা IP address সহ config file। এটা কাজ করে যতক্ষণ না inventory service 3টা instance থেকে 12টাতে autoscale করে, আর তাদের মধ্যে 9টা কোনো traffic-ই পায় না। অথবা এটা নতুন host-এ নতুন IP নিয়ে redeploy হয়, আর config stale হয়ে যায়। অথবা একটা instance মারা যায় আর caller-রা কেউ খেয়াল না করা পর্যন্ত একটা মৃত address-এ request পাঠাতেই থাকে।

তাই আপনি সামনে একটা load balancer বসান এবং তার address hardcode করেন। ভালো — কিন্তু এখন প্রতিটি service-এর জন্য একটা করে load balancer দরকার, প্রতিটাকে manually configure করতে হবে, নতুন service এলেই একটা করে provision করতে হবে। একটা Kubernetes cluster-এ, যেখানে pod ক্রমাগত তৈরি ও ধ্বংস হয়, manual configuration কোনো strategy নয়।

## এটি যে সমস্যাগুলো সমাধান করে

### ১. ক্রমাগত পরিবর্তনশীল address

**আপনি যা দেখেন:** মৃত instance-এ call, fleet জুড়ে অসম load, আর এমন deploy যার জন্য অন্য কয়েকটি service-এ configuration আপডেট করতে হয়।

**কেন এটা ঘটে:** একটা elastic, containerized environment-এ, instance address ডিজাইন অনুযায়ীই ephemeral (ক্ষণস্থায়ী)। যেকোনো static mapping লেখা মাত্রই stale হয়ে যায়।

**Service discovery কীভাবে এটা সমাধান করে:** Instance-গুলো startup-এ একটি registry-তে (Consul, etcd, ZooKeeper, বা Kubernetes-এর built-in service object) নিজেদের register করে এবং shutdown-এ deregister করে। Caller-রা নাম দিয়ে "inventory-service" lookup করে এবং বর্তমান healthy সেট পায়। Health check মৃত instance-গুলোকে স্বয়ংক্রিয়ভাবে বাদ দেয়। address সমস্যা মানুষের সমস্যা থাকে না আর।

### ২. ভুল communication style বেছে নেওয়া

**আপনি যা দেখেন:** একটা checkout flow যা নয়টি synchronous call করে এবং চার সেকেন্ড সময় নেয়, অথবা একটা inventory update asynchronously পাঠানো হয় যখন caller-এর সত্যিই জানার দরকার ছিল stock reserve হয়েছে কিনা।

**কেন এটা ঘটে:** দলগুলো একটা style বেছে নেয় — সাধারণত synchronous REST, কারণ এটা পরিচিত — আর সেটা সব জায়গায় প্রয়োগ করে।

**এই topic কীভাবে এটা সমাধান করে:** এটা case-গুলোকে আলাদা করে। **Synchronous** (REST, gRPC) যখন caller-এর এগিয়ে যাওয়ার জন্য উত্তর দরকার: pricing, authorization, stock check। **Asynchronous** (queues, events) যখন caller শুধু চায় কাজটা একসময় সম্পন্ন হোক: notifications, indexing, analytics, downstream প্রতিক্রিয়া। এই বিভাজন সঠিকভাবে করাই একটা responsive system আর একটা fragile system-এর মধ্যে বেশিরভাগ পার্থক্য তৈরি করে, কারণ প্রতিটি synchronous call তার latency যোগ করে এবং আপনার availability থেকে বিয়োগ করে।

### ৩. Chatty call যা latency-কে গুণিত করে

**আপনি যা দেখেন:** একটা মাত্র page render করতে 200টা internal call trigger হয় — এটা distributed version-এর N+1 query সমস্যা।

**কেন এটা ঘটে:** access pattern বিবেচনা না করে আঁকা service boundary caller-দের এমন remote call-এর ওপর loop করতে বাধ্য করে যা আগে একটা local join ছিল।

**এই topic কীভাবে এটা সমাধান করে:** এটা call-graph depth ও fan-out-কে একটা design concern বানায়। সমাধানের মধ্যে আছে batch endpoint, একটা BFF-এ aggregation, service boundary জুড়ে data duplication, এবং — সবচেয়ে গুরুত্বপূর্ণ — এমন boundary পুনর্বিবেচনা করা যেগুলোর জন্য একটা screen সন্তুষ্ট করতে remote call-এর একটা loop দরকার হয়।

### ৪. Internal volume-এ protocol overhead

**আপনি যা দেখেন:** লক্ষ লক্ষ ছোট internal call-এর জন্য JSON serialization ও HTTP header parsing-এ significant CPU খরচ করা service।

**কেন এটা ঘটে:** HTTP/1.1-এর ওপর JSON public API-এর জন্য চমৎকার — human-readable, universally supported, debuggable — আর high-volume internal traffic-এর জন্য তুলনামূলকভাবে ব্যয়বহুল।

**gRPC কীভাবে এটা সমাধান করে:** HTTP/2-এর ওপর binary Protocol Buffers ছোট payload, দ্রুততর serialization, একটা connection-এর ওপর multiplexed stream, আর দুই পাশেই একটা generated, type-checked contract দেয়। এর খরচ হলো human readability এবং সহজ `curl`-ability হারানো, আর এই কারণেই common pattern হলো edge-এ REST, ভেতরে gRPC।

## যে মূল্য আপনাকে দিতে হয়

- **Registry নিজেই critical infrastructure।** যদি service discovery down থাকে, তাহলে কিছুই কিছু খুঁজে পাবে না। এটা highly available হতে হবে (তাই এটা সাধারণত consensus-backed হয়) এবং client-দের সর্বশেষ known good সেট cache করতে হবে এবং gracefully degrade করতে হবে।
- **Client-side বনাম server-side discovery একটা প্রকৃত trade-off।** Client-side (caller registry query করে এবং নিজেই load balance করে) কার্যকর এবং একটা hop সরিয়ে দেয়, কিন্তু discovery logic প্রতিটি service ও প্রতিটি ভাষায় বসিয়ে দেয়। Server-side (একটা proxy এটা করে) logic-কে কেন্দ্রীভূত করে, কিন্তু একটা অতিরিক্ত hop-এর বিনিময়ে। Service mesh একটা sidecar proxy দিয়ে এটা সমাধান করে — আর পরিচালনার জন্য একটা সম্পূর্ণ control plane যোগ করে।
- **Registration মিথ্যা বলতে পারে।** যে instance আসলে প্রস্তুত হওয়ার আগেই register করে ফেলে সে এমন traffic পায় যা সে serve করতে পারে না; যে crash করে deregister না করেই, সে health check ধরে ফেলা পর্যন্ত একটা ভূতুড়ে entry রেখে যায়। Readiness বনাম liveness পার্থক্য ঠিক এই কারণেই বিদ্যমান।
- **প্রতিটি synchronous call একটা coupling।** এটা latency ও failure-কে ওপরের দিকে পাঠায়। এই কারণেই topic 26-এর pattern-গুলো এখানে ঐচ্ছিক নয়।
- **Versioning এখন একটা distributed contract সমস্যা।** একটা request বা response shape বদলানো এমন caller-দের ভাঙতে পারে যাদের ওপর আপনার নিয়ন্ত্রণ নেই, তাই আপনার backward-compatible evolution এবং একটা deprecation process দরকার।

## কখন এটা দরকার — আর কখন দরকার নেই

| যখন আপনার discovery দরকার | যখন আপনি এড়িয়ে যেতে পারেন |
|---|---|
| Instance ephemeral (containers, autoscaling) | নির্দিষ্ট host-এ হাতে গোনা কয়েকটা service |
| Service ঘন ঘন যোগ ও অপসারণ হয় | DNS প্লাস একটা static load balancer যথেষ্ট |
| আপনি Kubernetes বা একটা cloud orchestrator-এ চালান | in-process call-সহ একটা monolith |
| আপনার health-aware routing এবং failover দরকার | — |

| যখন Synchronous | যখন Asynchronous |
|---|---|
| Caller-এর এগিয়ে যাওয়ার জন্য ফলাফল দরকার | কাজটা পরে সম্পন্ন হতে পারে |
| Operation দ্রুত এবং নির্ভরযোগ্য | Downstream ধীর, bursty, বা অনির্ভরযোগ্য |
| Strong consistency প্রয়োজন | একাধিক consumer-এর একই event দরকার |

## এটা কেন Interview-এ আসে

একবার আপনি একটা microservices diagram এঁকে ফেললে, "এই service-গুলো একে অপরকে কীভাবে খুঁজে পায়?" এবং "এটা down থাকলে কী হবে?" প্রায় স্বয়ংক্রিয়ভাবেই আসে। Interviewer-রা চান service discovery-এর নাম শুনতে (এবং Kubernetes context-এ, এটা built-in তা), reason-সহ একটা ইচ্ছাকৃত synchronous/asynchronous বিভাজন, এবং প্রতিটি synchronous hop-এ প্রয়োগ করা resilience pattern। যে candidate বলে "আমি high-volume call-এর জন্য internally gRPC ব্যবহার করব এবং public API-এর জন্য edge-এ REST রাখব, যা caller-এর অপেক্ষা করার দরকার নেই তার জন্য event সহ" — সে ঠিক সঠিক মাত্রার চিন্তা প্রদর্শন করছে।

## এটা কীভাবে সংযুক্ত

এটা **microservices**-এর (topic 30) অপারেশনাল বাস্তবতা। Registry সাধারণত একটা **consensus**-backed store-এ (topic 27) চলে। Synchronous call-এর জন্য দরকার **circuit breakers এবং retries** (topic 26); asynchronous call-এর জন্য দরকার **queues** (topic 20), **Pub/Sub** (topic 21), এবং **idempotency** (topic 29)। Client-side load balancing প্রায়ই **consistent hashing** (topic 24) ব্যবহার করে। **API gateway** (topic 9) এই সবকিছুর বাহ্যিক মুখ, **gRPC এবং Protocol Buffers** কভার করা হয়েছে topic 33 ও 35-এ, এবং **Kubernetes** (topic 44) natively discovery প্রদান করে।

**পরবর্তী:** [Domain-Driven Design Basics](../32-domain-driven-design-basics/why.md) — কীভাবে ঠিক করবেন service boundary গুলো প্রথমেই কোথায় হওয়া উচিত ছিল।
