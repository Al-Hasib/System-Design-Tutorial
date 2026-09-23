# Containers & Orchestration: Docker & Kubernetes Fundamentals

**অসুবিধার মাত্রা (Difficulty):** Intermediate

## শেখার লক্ষ্যসমূহ (Learning Objectives)

- একটি container আসলে কী, এবং এটি virtual machine থেকে কীভাবে আলাদা তা ব্যাখ্যা করা।
- container image কী এবং এটি কীভাবে "works on my machine" সমস্যার সমাধান করে তা বর্ণনা করা।
- একটি orchestrator (Kubernetes) আসলে কী করে তা ব্যাখ্যা করা: container-দের জন্য scheduling, scaling, self-healing, এবং service discovery।
- মূল Kubernetes বিল্ডিং ব্লকগুলো — Pods, Deployments, Services — এবং প্রতিটির দায়িত্ব কী তা বর্ণনা করা।
- liveness এবং readiness probe ব্যাখ্যা করা এবং কেন এগুলো একটি container-এর প্রকৃত availability-র জন্য গুরুত্বপূর্ণ তা বোঝা।

## স্ক্রিপ্ট (Script)

### হুক / পরিচিতি (Hook / Intro)

এই কোর্স জুড়ে আমরা "server", "service", এবং "instance" নিয়ে কিছুটা বিমূর্তভাবে কথা বলেছি — একটি ডায়াগ্রামের বাক্স যা আপনার কোড চালায়। আধুনিক backend ইঞ্জিনিয়ারিংয়ে, সেই বাক্সটি প্রায় সবসময়ই একটি **container**, যা scheduled এবং পরিচালিত হয় একটি **orchestrator** দ্বারা — সাধারণত Kubernetes। এটি কোনো আকস্মিক ইনফ্রাস্ট্রাকচার খুঁটিনাটি বিষয় নয় — এটি সরাসরি নির্ধারণ করে যে load balancing, service discovery, এবং scaling (এই কোর্সে আগে আলোচিত সবকিছু) বাস্তবে কীভাবে বাস্তবায়িত হয়। আজ আমরা খতিয়ে দেখব একটি container আসলে কী, কেন এটি এই কাজের জন্য বিকল্প (virtual machine)-কে হারিয়ে দিয়েছে, এবং আপনি যখন Kubernetes-কে "just deploy this" বলেন তখন এটি আসলে কী করছে।

### Containers বনাম Virtual Machines

একটি **virtual machine** একটি সম্পূর্ণ কম্পিউটারকে virtualize করে, যার নিজস্ব পূর্ণাঙ্গ operating system kernel-সহ, যা একটি hypervisor-এর উপর চলে। এটি শক্তিশালী isolation দেয়, কিন্তু প্রতিটি VM একটি সম্পূর্ণ OS-এর overhead বহন করে — গিগাবাইট ডিস্ক, বাস্তব মেমরি overhead, এবং ধীর boot time (প্রায়ই কয়েক দশ সেকেন্ড থেকে মিনিট)। বিপরীতে একটি **container** operating system স্তরে virtualize করে: একই host-এ চলা container-গুলো host-এর OS kernel share করে, কিন্তু প্রতিটি নিজস্ব isolated filesystem, process namespace, এবং network stack পায়, যা Linux namespaces এবং cgroups-এর মতো kernel feature দ্বারা প্রয়োগ করা হয়। যেহেতু container-গুলো নিজস্ব kernel বহন করে না, তাই এগুলো বহুগুণ হালকা — গিগাবাইটের বদলে মেগাবাইট, এবং এগুলো মিনিটের বদলে মিলিসেকেন্ড থেকে কয়েক সেকেন্ডে শুরু হয়। এর isolation একটি পূর্ণাঙ্গ VM-এর চেয়ে দুর্বল (একটি kernel-স্তরের vulnerability তাত্ত্বিকভাবে container সীমানা অতিক্রম করতে পারে, যা VM সীমানা অতিক্রম করতে পারে না), কিন্তু আপনার নিজের বিশ্বস্ত application code-এর অনেকগুলো instance চালানোর জন্য, এই trade-off অত্যন্তভাবে container-এর পক্ষে যায়, এবং ঠিক এই কারণেই backend service-এর deployment-এর ডিফল্ট একক হয়ে উঠেছে container।

### Images: "Works on My Machine" সমস্যার সমাধান

একটি **container image** হলো একটি প্যাকেজড, অপরিবর্তনীয় (immutable) স্ন্যাপশট, যাতে থাকে আপনার application code, এর runtime (যেমন একটি নির্দিষ্ট Node.js বা Python সংস্করণ), এর library, এবং এর configuration — এটি চালানোর জন্য প্রয়োজনীয় সবকিছু, যা একটি `Dockerfile` থেকে একবার build করা হয় এবং যেখানেই একটি container runtime ইনস্টল করা আছে সেখানে অভিন্নভাবে চলে। এটি সরাসরি ক্লাসিক "works on my machine" সমস্যাটি খতম করে দেয়: যদি এটি আপনার laptop-এ image-এর ভেতরে সঠিকভাবে চলে, তাহলে এটি একজন সহকর্মীর laptop-এ, CI-তে, এবং production-এ ঠিক একইভাবে চলবে, কারণ এটি প্রতিবারই একই ফাইলসিস্টেম এবং dependency — "আশা করি একই সংস্করণ ইনস্টল হয়েছে" নয়। Image-গুলো সাধারণত একটি **registry**-তে সংরক্ষিত হয় (Docker Hub, বা একটি প্রাইভেট সমতুল্য) এবং layer-এ layer-এ build করা হয়, যেখানে অপরিবর্তিত layer-গুলো cache করা হয় এবং বিভিন্ন build জুড়ে পুনরায় ব্যবহার করা হয়, যা rebuild এবং image pull-কে দ্রুত রাখে।

### একটি Orchestrator আসলে কী করে

একটি machine-এ একটি container চালানো সহজ — কিন্তু দৃঢ়ভাবে (resiliently) কয়েক ডজন machine জুড়ে শত শত container চালানো হলো আসল কঠিন সমস্যা, এবং এটিই **Kubernetes** (বা অনুরূপ একটি orchestrator) সমাধান করে। নির্দিষ্টভাবে বললে, একটি orchestrator যা পরিচালনা করে তা হলো: **scheduling** — উপলব্ধ resource-এর ভিত্তিতে প্রতিটি container instance আসলে কোন physical machine-এ চলবে তা ঠিক করা; **scaling** — load-এর উপর ভিত্তি করে স্বয়ংক্রিয়ভাবে container instance যোগ বা বাদ দেওয়া (Module 1-এর horizontal scaling ধারণার সরাসরি বাস্তবায়ন); **self-healing** — একটি crash হওয়া container স্বয়ংক্রিয়ভাবে পুনরায় চালু করা, এবং যে machine-এ এটি চলছিল সেটি সম্পূর্ণরূপে fail করলে অন্য কোথাও পুনরায় schedule করা; **service discovery এবং load balancing** — container instance-এর একটি গ্রুপকে একটি স্থিতিশীল নাম/ঠিকানা দেওয়া যাতে অন্য service-গুলো "payments service"-এ পৌঁছাতে পারে, বর্তমানে কোন নির্দিষ্ট instance বিদ্যমান বা কোথায় চলছে তা জানার বা চিন্তা করার প্রয়োজন ছাড়াই (Module 7 থেকে service discovery স্মরণ করুন); এবং **rolling updates** — পুরনো container instance-গুলোকে ধীরে ধীরে নতুন দিয়ে প্রতিস্থাপন করা, যা পরবর্তী ভিডিওতে আলোচিত zero-downtime deployment strategy-র প্রকৃত মেকানিজম।

### Kubernetes-এর মূল বিল্ডিং ব্লক

Kubernetes এই সবকিছুকে একটি ছোট সেট মূল object-এর চারপাশে সংগঠিত করে। একটি **Pod** হলো ক্ষুদ্রতম deployable একক — এক বা একাধিক নিবিড়ভাবে সংযুক্ত (tightly-coupled) container যা সবসময় একসাথে একই machine-এ schedule হয়, একটি network namespace share করে (বাস্তবে সাধারণত পড-প্রতি একটিই application container থাকে)। একটি **Deployment** pod-এর একটি সেটের *কাঙ্ক্ষিত অবস্থা (desired state)* বর্ণনা করে — "আমি চাই এই container image-এর 5টি replica সবসময় চলমান থাকুক" — এবং Kubernetes ক্রমাগত চেষ্টা করে যাতে বাস্তবতা সেই কাঙ্ক্ষিত অবস্থার সাথে মেলে, প্রয়োজন অনুযায়ী pod পুনরায় চালু করে বা পুনরায় schedule করে সংখ্যা বজায় রাখতে। একটি **Service** সেই স্থিতিশীল network পরিচয় প্রদান করে যা আমরা মাত্র উল্লেখ করেছি — একটি একক, অপরিবর্তনীয় ঠিকানা যা যেসব pod বর্তমানে healthy এবং একটি Deployment-এর অংশ তাদের কাছে route করে, এমনকি যখন পৃথক pod-গুলো প্রতিস্থাপিত, পুনরায় schedule, বা এর নিচে scale up/down হয় — একটি অভ্যন্তরীণ load balancer হিসেবে কাজ করে।

Kubernetes-এর এটাও জানা দরকার যে একটি pod ট্রাফিক গ্রহণের জন্য আসলেই যথেষ্ট healthy কিনা, এবং এখানেই আসে **liveness এবং readiness probe** — orchestrator প্রতিটি pod-এর বিরুদ্ধে যে পর্যায়ক্রমিক health check চালায়। একটি **liveness probe** উত্তর দেয় "এই container কি এখনও জীবিত, নাকি এটিকে kill করে পুনরায় চালু করা উচিত?" (একটি hung বা deadlocked প্রসেস ধরার জন্য)। একটি **readiness probe** উত্তর দেয় "এই container কি এই মুহূর্তে ট্রাফিক গ্রহণ করতে প্রস্তুত?" (একটি pod হয়তো জীবিত কিন্তু এখনও একটি cache warm up করছে বা কোনো dependency-র জন্য অপেক্ষা করছে, এবং এখনই request গ্রহণ করা উচিত নয়) — একটি pod যার readiness probe ব্যর্থ হয় তা স্বয়ংক্রিয়ভাবে একটি Service-এর routing থেকে সরিয়ে দেওয়া হয় যতক্ষণ না এটি আবার pass করে, liveness probe ব্যর্থ হলে যেভাবে restart trigger হতো সেভাবে restart না করেই।

### বাস্তব-জগতের উদাহরণ (Real-World Example)

আমাদের বারবার ব্যবহৃত payments microservice উদাহরণটি deploy করার কথা বিবেচনা করুন। আপনি service এবং এর সঠিক dependency-সহ একটি Docker image build করেন, এটি একটি registry-তে push করেন, এবং 6টি replica চেয়ে একটি Kubernetes Deployment সংজ্ঞায়িত করেন। Kubernetes আপনার উপলব্ধ machine জুড়ে সেই 6টি pod schedule করে, এবং একটি Service আপনার সিস্টেমের বাকি অংশকে একটি স্থিতিশীল ঠিকানা দেয় — `payments-service` — কল করার জন্য, বর্তমানে কোন নির্দিষ্ট pod চলছে বা কোথায় চলছে তা নির্বিশেষে। যদি ট্রাফিক বেড়ে যায়, একটি autoscaler (একই scaling মেকানিজমের উপর নির্মিত) replica সংখ্যা বাড়িয়ে 12 করে; যদি অন্তর্নিহিত machine-গুলোর একটি crash করে, Kubernetes লক্ষ্য করে যে এটি যে pod-গুলো চালাচ্ছিল সেগুলো চলে গেছে এবং সেগুলো অন্য কোথাও পুনরায় schedule করে, কোনো মানুষের হস্তক্ষেপ ছাড়াই কাঙ্ক্ষিত 12টি replica পুনরুদ্ধার করে। এবং যখন একটি নতুন সংস্করণ deploy করা প্রয়োজন হয়, readiness probe নিশ্চিত করে যে একটি নতুন pod ততক্ষণ live ট্রাফিক গ্রহণ করবে না যতক্ষণ না এটি সত্যিই প্রস্তুত নিশ্চিত হয় — সরাসরি "new instance আসলে initialize হওয়ার আগেই ট্রাফিক পায়" এই failure mode প্রতিরোধ করে।

### সংক্ষিপ্তকরণ (Recap)

Container-গুলো OS স্তরে virtualize করে, host kernel share করে কিন্তু filesystem, process, এবং network isolate করে — পূর্ণাঙ্গ virtual machine-এর তুলনায় বহুগুণ হালকা এবং দ্রুত, সামান্য দুর্বল isolation-এর বিনিময়ে। Container image একটি application এবং এর প্রয়োজনীয় সবকিছুকে একটি অপরিবর্তনীয়, বহনযোগ্য (portable) একক-এ প্যাকেজ করে, চিরকালের জন্য "works on my machine" সমাধান করে। Kubernetes (একটি orchestrator) অনেক machine জুড়ে অনেক container-এর scheduling, scaling, self-healing, service discovery, এবং rolling update পরিচালনা করে — Pods (ক্ষুদ্রতম deployable একক), Deployments (কাঙ্ক্ষিত-অবস্থা পরিচালনা), এবং Services (স্থিতিশীল network পরিচয়) কে ঘিরে সংগঠিত, যেখানে liveness এবং readiness probe নিশ্চিত করে যে শুধুমাত্র প্রকৃতপক্ষে healthy pod-গুলোই ট্রাফিক গ্রহণ করে।

### এরপর কী (What's Next)

এখন যেহেতু আমরা জানি container-গুলো কীভাবে schedule করা হয় এবং সুস্থ রাখা হয়, পরবর্তী ভিডিওতে ঠিক কীভাবে একটি নতুন সংস্করণ downtime ছাড়াই production-এ পুরনোটিকে প্রতিস্থাপন করে তা কভার করা হবে — যা সরাসরি আমরা মাত্র পরিচয় করিয়ে দেওয়া rolling-update এবং readiness-probe মেকানিজমের উপর নির্মিত — এর পাশাপাশি একটি সিস্টেমের নিচে database-এর schema পরিবর্তনের আরও জটিল সমস্যা, যেটি চলা বন্ধ করতে পারে না।

## মূল বার্তাসমূহ (Key Takeaways)

- Container-গুলো OS স্তরে virtualize করে (host kernel share করে, namespaces/cgroups-এর মাধ্যমে isolate করে), যা এগুলোকে পূর্ণাঙ্গ virtual machine-এর তুলনায় অনেক বেশি হালকা এবং দ্রুত শুরু হওয়ার উপযোগী করে তোলে, কিছুটা দুর্বল isolation-এর বিনিময়ে।
- একটি container image code, runtime, এবং dependency-কে একটি অপরিবর্তনীয়, বহনযোগ্য এককে প্যাকেজ করে যা একবার build হয় এবং যেকোনো জায়গায় অভিন্নভাবে চলে — "works on my machine" সমাধান করে।
- Kubernetes (একটি orchestrator) অনেক machine জুড়ে container-দের জন্য scheduling, scaling, self-healing, service discovery, এবং rolling update পরিচালনা করে।
- Pod হলো ক্ষুদ্রতম deployable একক; Deployment কাঙ্ক্ষিত replica অবস্থা পরিচালনা করে; Service একটি স্থিতিশীল network পরিচয় প্রদান করে যা healthy pod জুড়ে load-balance করে।
- Liveness probe ঠিক করে একটি hung container পুনরায় চালু করা হবে কিনা; readiness probe ঠিক করে একটি pod বর্তমানে ট্রাফিক গ্রহণ করা উচিত কিনা — ভিন্ন সমস্যার জন্য ভিন্ন check।
</content>
