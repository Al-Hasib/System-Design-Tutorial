# এই বিষয়টি কেন গুরুত্বপূর্ণ: Containers & Orchestration (Docker & Kubernetes)

> **এক বাক্যে:** Container-গুলো "it works on my machine"-কে সর্বত্র সত্য করে তোলে, এবং orchestration server-এর একটি বহরকে একক capacity pool-এ পরিণত করে যা কারো নজরদারি ছাড়াই আপনার service চালু রাখে।

## এই ধারণার আগের পৃথিবী (The World Before This Idea)

**Container-এর আগে।** একটি deploy মানে হলো কোডকে একটি server-এ কপি করা এবং আশা করা যে এর environment মিলে যাবে। এটি মেলে না: production-এ Python 3.8 আছে আর developer-এর কাছে আছে 3.11; একটি system library এক সংস্করণ পিছিয়ে; একটি environment variable অনুপস্থিত। Environment drift ডিবাগ করাটা feature লেখার চেয়ে বেশি সময় খরচ করে। একই host-এর দুটি application-এর একই dependency-র বেমানান সংস্করণ প্রয়োজন হয়, তাই আপনি প্রত্যেককে তার নিজস্ব server দেন — এবং এখন আপনি 5% utilization-এ বিশটি machine চালাচ্ছেন কারণ সেগুলোকে isolate করার এটাই একমাত্র উপায় ছিল।

**Orchestration-এর আগে।** আপনার কাছে container আছে, যা ভালো। এখন আপনার কাছে বারোটি host জুড়ে ষাটটি container আছে। পরবর্তী container-এর জন্য কোন host-এ capacity আছে? রাত ২টায় একটি container crash করে — কে এটা restart করবে? একটি host মারা যায় — কে এর workload স্থানান্তর করবে? একটি ট্রাফিক স্পাইকের জন্য scale up করা মানে কাউকে ম্যানুয়ালি container চালু করতে হবে এবং ম্যানুয়ালি load balancer configuration আপডেট করতে হবে। এটি একটি পূর্ণকালীন চাকরি, যা মানুষ দ্বারা, মানুষের গতিতে এবং মানুষের নির্ভরযোগ্যতায় সম্পন্ন হয়।

## এটি যে সমস্যাগুলো সমাধান করে (The Problems It Solves)

### ১. Development, CI, এবং production-এর মধ্যে environment drift

**আপনি যা দেখেন:** এমন কোড যা test pass করে কিন্তু production-এ ব্যর্থ হয়, কোডের সাথে সম্পর্কহীন কারণে।

**কেন এটি ঘটে:** runtime environment প্রতিটি জায়গায় আলাদাভাবে, ভিন্ন মেকানিজম দ্বারা তৈরি হয়, এবং drift করে।

**Container যেভাবে এটি সমাধান করে:** image application-কে তার সম্পূর্ণ userland-সহ bundle করে — library, runtime, system package, configuration। যে artifact test pass করেছে সেটিই বিট-বাই-বিট সেই artifact যা production-এ চলে। এই একটি বৈশিষ্ট্যই ইনসিডেন্টের একটি সম্পূর্ণ ক্যাটাগরি নির্মূল করে এবং এটিই কারণ যে container-গুলো জিতে গেছে।

### ২. Server-প্রতি-application isolation থেকে দুর্বল utilization

**আপনি যা দেখেন:** ডজন ডজন বেশিরভাগ idle machine, স্বতন্ত্রভাবে provisioned কারণ application-গুলো নিরাপদে একটি host share করতে পারে না।

**কেন এটি ঘটে:** process-স্তরের isolation ছাড়া, application-গুলো dependency, port, এবং resource নিয়ে সংঘর্ষে জড়ায়।

**Container যেভাবে এটি সমাধান করে:** Namespaces এবং cgroups একটি virtual machine-এর খরচের ভগ্নাংশে isolation দেয় — কোনো guest OS নেই, শুরু হয় মিনিটের বদলে মিলিসেকেন্ডে, প্রতি container-এ CPU এবং memory limit প্রয়োগ করা হয়। অনেক workload নিরাপদে একটি host-এ প্যাক করা যায়, এবং utilization একক অঙ্ক থেকে একটি সমর্থনযোগ্য মাত্রায় পৌঁছায়।

### ৩. ম্যানুয়াল অপারেশন যা স্কেল করে না এবং রাত ৩টায় ঘটে না

**আপনি যা দেখেন:** crash হওয়া process যা কেউ লক্ষ্য না করা পর্যন্ত down থাকে। Scaling যার জন্য একজন মানুষ প্রয়োজন। একটি মৃত host যা drain করতে ঘন্টার পর ঘন্টা লাগে কারণ কারো কাছে কোনো runbook নেই।

**কেন এটি ঘটে:** কোনো control loop নেই — শুধু মানুষ আছে।

**Kubernetes যেভাবে এটি সমাধান করে:** আপনি কাঙ্ক্ষিত অবস্থা ঘোষণা করেন ("এই image-এর পাঁচটি replica, এই resource limit-সহ, এই service-এর পেছনে"), এবং controller-গুলো ক্রমাগত বাস্তবতাকে সেদিকে reconcile করে। একটি container মারা গেলে, এটি restart হয়। একটি host মারা গেলে, এর pod-গুলো অন্য কোথাও পুনরায় schedule হয়। Health check স্বয়ংক্রিয়ভাবে অসুস্থ pod-গুলোকে service থেকে সরিয়ে দেয়। *imperative steps* থেকে *reconciliation loop-সহ declarative desired state*-এ এই পরিবর্তনটিই মূল ধারণা, এবং এটিই অপারেশনকে ম্যানুয়াল শ্রম থেকে একটি সিস্টেমে রূপান্তরিত করে।

### ৪. Deployment, discovery, config, এবং scaling চারটি পৃথক সমস্যা হিসেবে

**আপনি যা দেখেন:** হাতে scripted rolling update, config file-এ পরিচালিত service address, image-এ বেক করা secret, এবং cloud API থেকে একত্রে জোড়া দেওয়া autoscaling।

**কেন এটি ঘটে:** প্রতিটি সমস্যা তার নিজস্ব tooling দিয়ে আলাদাভাবে সমাধান করা হয়।

**Kubernetes যেভাবে এটি সমাধান করে:** স্বয়ংক্রিয় rollback-সহ rolling update, DNS-এর মাধ্যমে built-in service discovery, configuration-এর জন্য ConfigMap এবং Secret, এবং metric-ভিত্তিক horizontal pod autoscaling — সবকিছুই একই system থেকে আসে একই declarative মডেল দিয়ে। সবকিছুর জন্য একটি সুসংগত মডেল থাকাটা যেকোনো একক feature-এর চেয়ে বেশি মূল্যবান।

## আপনি যে মূল্য পরিশোধ করেন (The Price You Pay)

Kubernetes শক্তিশালী এবং সত্যিকার অর্থে ভারী, এবং এটাই সৎ অংশ:

- **শেখার বক্ররেখা খাড়া এবং প্রশস্ত।** Pod, Deployment, Service, Ingress, ConfigMap, Secret, StatefulSet, DaemonSet, PVC, RBAC, network policy, admission controller। দলগুলো নিয়মিতভাবে এটিকে একটি মাত্রার ক্রম দ্বারা অবমূল্যায়ন করে।
- **এর জন্য নিবেদিত দক্ষতা প্রয়োজন।** Upgrade, networking (CNI), storage (CSI), এবং certificate rotation বিশেষজ্ঞের কাজ। যে cluster-এর কেউ মালিক নয় সেটি একটি দায়ে পরিণত হয়।
- **ডিবাগিং আরও বেশি স্তর জুড়ে বিস্তৃত।** একটি failure application-এ, container-এ, pod spec-এ, service-এ, ingress-এ, network policy-তে, অথবা node-এ থাকতে পারে। রোগনির্ণয়ের জন্য এগুলো সবকিছু বোঝা প্রয়োজন।
- **Stateful workload কঠিন।** Kubernetes-এ database চালানো সম্ভব এবং একটি managed database service-এর চেয়ে উল্লেখযোগ্যভাবে বেশি জটিল। বেশিরভাগ দলের জন্য, cluster-এর ভেতরে Postgres চালানো এমন একটি পছন্দ যার জন্য একটি শক্তিশালী যুক্তির প্রয়োজন।
- **ভুল configuration outage ঘটায়।** ভুল resource limit OOM kill এবং eviction ঘটায়; একটি খারাপ readiness probe একটি healthy service-কে rotation থেকে বের করে দেয়; একটি সীমাহীন request একটি node-কে অভুক্ত রাখতে পারে।
- **এটি প্রায়ই অতিরিক্ত (overkill)।** তিনটি service চালানো একটি দল একটি managed platform-এ (ECS, Cloud Run, App Runner, Fly, Render, Heroku-style) দ্রুত এগোবে যা একই বিষয়গুলো অনেক কম পরিসরের সারফেসে পরিচালনা করে। Kubernetes তার খরচ আদায় করে স্কেল এবং বৈচিত্র্য দিয়ে, উচ্চাকাঙ্ক্ষা দিয়ে নয়।

## কখন এটি প্রয়োজন — এবং কখন নয় (When You Need It — and When You Don't)

| Kubernetes যখন                                            | সরল কিছু যখন                                  |
| ---------------------------------------------------------- | --------------------------------------------- |
| অনেক service, অনেক দল, ঘন ঘন deploy                        | কয়েকটি service এবং একটি ছোট দল                |
| আপনার multi-cloud বা on-prem portability প্রয়োজন            | আপনি এক cloud-এর managed platform-এ খুশি      |
| Workload-গুলো ভিন্নধর্মী (heterogeneous), scaling profile বিভিন্ন | Workload একরকম এবং পূর্বাভাসযোগ্য              |
| আপনার platform দক্ষতা আছে বা নিয়োগ দিতে পারেন                 | কেউ পূর্ণকালীনভাবে ইনফ্রাস্ট্রাকচারের মালিক নয় |

**তবে container-গুলো প্রায় সর্বজনীনভাবে মূল্যবান** — এমনকি যদি আপনি সেগুলো একটি managed platform-এ deploy করেন এবং কখনও কোনো orchestrator স্পর্শ না করেন।

## এটি ইন্টারভিউতে কেন আসে (Why This Shows Up in Interviews)

Deployment প্রায়ই একটি design আলোচনার সমাপ্তি হয়: "আপনি এটি কীভাবে চালাবেন এবং স্কেল করবেন?" আপনার কাছে Kubernetes object মুখস্থ করে বলার আশা করা হয় না। আপনার কাছে আশা করা হয় যে আপনি ব্যাখ্যা করতে পারবেন service কীভাবে প্যাকেজ করা হয়, instance কীভাবে scale up/down করা হয়, একটি মারা গেলে কী ঘটে, একটি নতুন সংস্করণ নিরাপদে কীভাবে rollout করা হয়, এবং service-গুলো একে অপরকে কীভাবে খুঁজে পায় — এবং জানতে হবে যে একটি orchestrator এই সবকিছুই একটি সিস্টেম হিসেবে প্রদান করে। যেসব প্রার্থী আরও বলেন "এই আকারের সিস্টেমের জন্য, একটি managed container platform যথেষ্ট হবে এবং Kubernetes অতিরিক্ত হবে" তারা সেই judgment প্রদর্শন করেন যা ইন্টারভিউয়াররা আসলে যাচাই করছেন।

## এটি কীভাবে সংযুক্ত (How It Connects)

Orchestration হলো যেভাবে **horizontal scaling** (বিষয় ৪) এবং **fault tolerance** (বিষয় ৫) বাস্তবে বাস্তবায়িত হয়। এটি নেটিভভাবে **service discovery** (বিষয় ৩১) এবং Service ও Ingress-এর মাধ্যমে **load balancing** (বিষয় ৭) প্রদান করে। এর control plane **etcd**-এ state সংরক্ষণ করে এবং **consensus** ও leader election ব্যবহার করে (বিষয় ২৭, ৪০)। Rolling update এবং readiness probe হলো **zero-downtime deployment**-এর (বিষয় ৪৫) যন্ত্রপাতি, এবং zone জুড়ে pod scheduling হলো **multi-region architecture**-এর (বিষয় ৪৭) একটি বিল্ডিং ব্লক।

**পরবর্তী:** [Zero-Downtime Deployments &amp; Database Migrations](../45-zero-downtime-deployments-and-database-migrations/why.md) — কারো লক্ষ্য না করেই পরিবর্তন শিপ করা।
</content>
