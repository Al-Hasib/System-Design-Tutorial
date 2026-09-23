# Microservices Communication ও Service Discovery

**Difficulty:** Advanced

## Learning Objectives

এই ভিডিও শেষ করার পর আপনি নিচের বিষয়গুলো পারবেন:

- synchronous (REST/gRPC) এবং asynchronous (message-queue/event-based) inter-service communication-এর মধ্যে পার্থক্য বুঝতে পারা, এবং কখন কোনটা ব্যবহার করতে হবে তা জানা।
- service discovery কোন সমস্যার সমাধান করে এবং dynamic environment-এ static configuration কেন ভেঙে পড়ে তা ব্যাখ্যা করা।
- client-side discovery, server-side discovery, এবং service-mesh-based discovery-এর মধ্যে তুলনা করা।
- একটি service registry (যেমন Consul বা Eureka) কীভাবে কাজ করে তা বর্ণনা করা, যার মধ্যে health check এবং registration/deregistration অন্তর্ভুক্ত।
- একটি প্রকৃত request path-এ API gateway, load balancer, এবং service discovery কোথায় একসাথে কাজ করে তা চিহ্নিত করা।

## Script

### Hook / Intro

কল্পনা করুন, আপনি সবেমাত্র আপনার monolith-কে বিশটি microservices-এ ভেঙেছেন, ঠিক যেমনটা আমরা আগের ভিডিওতে আলোচনা করেছিলাম। অভিনন্দন — এখন order service-কে inventory service কল করতে হবে stock চেক করার জন্য। একটা ছোট্ট প্রশ্ন: এটা কোন IP address কল করবে? auto-scaling group, Kubernetes-এর মতো container orchestrator, এবং ক্রমাগত তৈরি ও ধ্বংস হওয়া instance-এর জগতে, সেই "IP address" প্রতি কয়েক মিনিটে বদলে যায়। এটা hardcode করা শুধু অসুবিধাজনক নয়, কাঠামোগতভাবে অসম্ভব। এটাই সেই সমস্যা যা সমাধানের জন্য service discovery-এর অস্তিত্ব, আর এর সাথে জড়িয়ে আছে একই রকম গুরুত্বপূর্ণ আরেকটি প্রশ্ন: কাকে কল করতে হবে তা জানার পর, service-গুলো আসলে একে অপরের সাথে কীভাবে কথা বলবে? চলুন দুটোই খতিয়ে দেখি।

### Communication Styles: Synchronous বনাম Asynchronous

Inter-service communication-এর দুটি বড় পরিবার আছে।

**Synchronous, request/response communication** — মনে করুন HTTP-এর ওপর REST, অথবা বেশি performance-sensitive, strongly-typed internal call-এর জন্য gRPC। order service, inventory service-কে কল করে এবং response-এর জন্য অপেক্ষায় block হয়ে থাকে: "এই item-টা কি stock-এ আছে, হ্যাঁ নাকি না?" এটা স্বজ্ঞাত (intuitive) এবং function call সম্পর্কে আমাদের চিন্তাভাবনার সাথে মেলে, এবং caller-এর যখন সত্যিই এগিয়ে যাওয়ার জন্য তাৎক্ষণিক উত্তর দরকার তখন এটা স্বাভাবিকভাবেই মানানসই। এর খরচ হলো coupling: যদি inventory service ধীর হয়ে যায় বা down হয়ে যায়, তাহলে order service-ও এখন ধীর বা down হয়ে যাবে, যদি না আপনি Module 6-এর timeout, retry, এবং circuit breaker pattern দিয়ে কলটিকে রক্ষা করেন।

**Asynchronous, event-driven communication** — order service একটি "OrderPlaced" event publish করে একটি message queue বা broker-এ, যেমন Kafka বা RabbitMQ, যা আমরা Module 5-এ আলোচনা করেছি, এবং অপেক্ষা না করেই এগিয়ে যায়। inventory service, notification service, এবং analytics service — সবাই সেই event-এ subscribe করে এবং নিজ নিজ সময়ে, স্বাধীনভাবে প্রতিক্রিয়া জানায়। এটা সময়ের দিক থেকে service-গুলোকে সম্পূর্ণভাবে decouple করে দেয় — যদি notification service পাঁচ মিনিটের জন্য down থাকে, event-গুলো শুধু queue-তে জমা হতে থাকবে এবং সেটা recover হলে process হবে, কিছুই হারিয়ে যাবে না, এবং order service কখনোই এর জন্য অপেক্ষায় block হয়নি।

ব্যবহারিক নিয়ম হলো: বর্তমান operation চালিয়ে যাওয়ার জন্য যখন আপনার এখনই একটা উত্তর দরকার — যেমন purchase নিশ্চিত করার আগে real-time stock চেক করা — তখন synchronous call ব্যবহার করুন। আর যা কিছু আগে ঘটে যাওয়া কোনো কিছুর প্রতিক্রিয়া এবং যেটার জন্য মূল flow-কে block করার প্রয়োজন নেই — যেমন confirmation email পাঠানো, recommendation model আপডেট করা, audit trail লগ করা — তার জন্য asynchronous event ব্যবহার করুন। বেশিরভাগ real system দুটোরই মিশ্রণ ব্যবহার করে।

### সমস্যা: কেন আপনার Service Discovery দরকার

একটা static জগতে, আপনি inventory service-এর address একটা config file-এ রেখে দিতেন আর কাজ শেষ। কিন্তু বাস্তব microservice deployment-গুলো dynamic: load-এর ভিত্তিতে instance বাড়ে-কমে, unhealthy instance মেরে ফেলে নতুন দিয়ে replace করা হয়, deployment ক্রমাগত নতুন version আনে ও পুরনো version সরায়, আর Kubernetes-এ একটা pod-এর IP address মূলত disposable এবং প্রতি restart-এ বদলে যায়। যদি order service IP hardcode করত, তাহলে সেটা ক্রমাগত ভেঙে পড়ত। Service discovery হলো সেই mechanism যার মাধ্যমে একটি service runtime-এ জিজ্ঞাসা করতে পারে, "আমাকে inventory service-এর একটা healthy address দাও" — এবং প্রতিবারই একটা সঠিক, up-to-date উত্তর পায়।

### Service Discovery-এর Pattern-গুলো

তিনটি common pattern আছে, আর তিনটিই জানা দরকার কারণ বাস্তব ক্ষেত্রে আপনি তিনটিরই দেখা পাবেন।

**Client-side discovery।** কল করা service — অর্থাৎ client — সরাসরি একটি service registry-কে query করে target service-এর healthy instance-এর একটি তালিকা পায়, তারপর নিজেই একটা বেছে নেয়, প্রায়ই round robin-এর মতো client-side load-balancing algorithm ব্যবহার করে। Netflix-এর Eureka, তাদের Ribbon client-side load balancer-এর সাথে মিলিয়ে, এই pattern-এর পাঠ্যপুস্তকীয় উদাহরণ। এর সুবিধা হলো load-balancing সিদ্ধান্তের ওপর client-এর সম্পূর্ণ দৃশ্যমানতা ও flexibility থাকে। অসুবিধা হলো প্রতিটি ভাষায় থাকা প্রতিটি client-এর মধ্যে discovery-aware logic বেক করে রাখতে হয়, যা একটি polyglot fleet জুড়ে একটা maintenance burden তৈরি করে।

**Server-side discovery।** client শুধু একটি single, stable endpoint কল করে — একটি load balancer বা router — আর সেই intermediary-ই registry-কে query করে এবং request-কে একটি healthy instance-এ forward করে। AWS ELB, ECS service discovery-এর সাথে মিলিয়ে, অথবা Kubernetes-এর built-in Service abstraction ও kube-proxy — এগুলো ক্লাসিক উদাহরণ: আপনার pod শুধু `inventory-service.default.svc.cluster.local` কল করে, আর Kubernetes এটিকে transparently পর্দার আড়ালে একটি healthy pod-এ route করে দেয়। এটা client application-এর জন্য সহজ — তাদের কোনো discovery logic-ই দরকার হয় না — কিন্তু এটা একটা network hop যোগ করে এবং load balancer-কে নিজেই একটা critical infrastructure বানিয়ে ফেলে।

**Service mesh discovery।** Istio বা Linkerd-এর মতো একটি service mesh প্রতিটি service instance-এর পাশে একটি লাইটওয়েট proxy — একটি "sidecar," সাধারণত Envoy — deploy করে। একটি service-এর সমস্ত ভেতরে-বাইরে যাওয়া network traffic তার sidecar-এর মধ্য দিয়ে যায়, আর sidecar-গুলো discovery, load balancing, retries, timeouts, mutual TLS encryption, এবং observability transparently সামলায়, একেবারে আপনার application code-এর বাইরে থেকে। এটা সবচেয়ে শক্তিশালী এবং operationally সবচেয়ে জটিল option, আর এটাই সেই জিনিস যা খুব বড়, mature microservice deployment-গুলো গ্রহণ করে থাকে কারণ এটা cross-cutting concern-গুলোকে প্রতি service-এ আলাদাভাবে reimplement না করে কেন্দ্রীভূত করে।

### একটি Service Registry আসলে কীভাবে কাজ করে

pattern যাই হোক না কেন, সাধারণত একটা কেন্দ্রীয় component থাকে: service registry — Consul, Eureka, অথবা etcd/Kubernetes-এর internal API server এই ভূমিকায় কাজ করে। এর lifecycle এরকম দেখতে: যখন একটি নতুন service instance চালু হয়, এটি নিজেকে registry-তে register করে, ঘোষণা করে "আমি inventory-service, আমি এই address-এ আছি, এই যে আমার health check endpoint।" এরপর registry পর্যায়ক্রমে সেই health check কল করে — সাধারণত একটা সাধারণ `/health` HTTP endpoint — আর যদি কোনো instance সাড়া দেওয়া বন্ধ করে দেয় বা unhealthy রিপোর্ট করে, registry সেটাকে down বলে চিহ্নিত করে এবং callers-দের কাছে সেই address দেওয়া বন্ধ করে দেয়। যখন কোনো instance স্বাভাবিকভাবে (gracefully) বন্ধ হয়, এটি নিজেকে deregister করে; যদি এটা crash করে, তাহলে ব্যর্থ health check শেষ পর্যন্ত সেটা ধরে ফেলে। registration, health checking, এবং deregistration-এর এই সমন্বয়ই নিশ্চিত করে যে callers-রা যে address-গুলো পায় সেগুলো সবসময় এমন instance-এর দিকেই নির্দেশ করে যা আসলেই traffic serve করতে পারে।

### বাস্তব-জগতের উদাহরণ

এই পুরো topic-এর জন্য Netflix হলো আদর্শ case study। তারা হাজার হাজার service instance চালায় যেগুলো traffic-এর সাথে ক্রমাগত বাড়ে-কমে। তারা Eureka বিশেষভাবে এমনভাবে তৈরি করেছিল যাতে যেকোনো service জিজ্ঞাসা করতে পারে "recommendation service-এর এখন healthy instance কারা?" এবং একটা live উত্তর পেতে পারে, সাথে client-side load balancing-এর জন্য Ribbon এবং resilience-এর জন্য Hystrix — যা আজকের circuit breaker library-গুলোর পূর্বপুরুষ। আজকাল, অনেক দল Kubernetes-এ চালিয়ে একই ফলাফল আরও সহজভাবে অর্জন করে, যা server-side service discovery-কে সরাসরি তার Service ও DNS abstraction-এর মধ্যে বেক করে রাখে, অথবা service-গুলোর মধ্যে traffic management ও security-এর ওপর আরও নিয়ন্ত্রণের জন্য এর ওপর Istio-এর মতো একটি service mesh layer করে।

### Recap

সংক্ষেপে বলতে গেলে: microservices একে অপরের সাথে হয় synchronously কথা বলে, যখন caller-এর তাৎক্ষণিক উত্তর দরকার, অথবা event ও queue-এর মাধ্যমে asynchronously, যখন তাদের শুধু ঘটে যাওয়া কোনো কিছুর প্রতিক্রিয়া জানাতে হয়। যেহেতু instance-গুলো ক্রমাগত আসে ও যায়, hardcoded address কাজ করে না — আপনার service discovery দরকার, যা implement করা যায় client-side discovery (caller registry-কে query করে এবং একটা instance বেছে নেয়), server-side discovery (একটা load balancer caller-এর হয়ে সেটা করে), অথবা একটা পূর্ণাঙ্গ service mesh যেখানে sidecar proxy transparently সেটা সামলায়। এই তিনটির নিচে থাকে একটি service registry যা ক্রমাগত health checking করে যাতে শুধুমাত্র healthy instance-গুলোই কখনো ফেরত দেওয়া হয়।

### এরপর কী

আমরা এখন কভার করেছি কীভাবে service structure করতে হয় এবং কীভাবে তারা একে অপরকে খুঁজে পায় ও কথা বলে। কিন্তু একটা গভীর প্রশ্ন আছে যেটার চারপাশে আমরা এই পুরো module জুড়ে ঘুরছিলাম: আপনি আসলে কীভাবে ঠিক করবেন একটি service কোথায় শেষ হয় আর আরেকটি কোথায় শুরু হয়? কী নির্ধারণ করে যে "inventory" এবং "orders" প্রথমেই আলাদা service হওয়া উচিত, "users" ও "billing"-এর বদলে? এটা একটা design সমস্যা, শুধু infrastructure সমস্যা নয়, আর পরের ভিডিওতে আমরা Domain-Driven Design পরিচয় করাব — একটি পদ্ধতি যা arbitrary technical বিভাজনের বদলে real business concept-এর চারপাশে সেই সীমারেখা আঁকার জন্য।

## Key Takeaways

- caller-এর তাৎক্ষণিক উত্তর দরকার হলে synchronous call (REST/gRPC) ব্যবহার করুন; একটি service-কে শুধু ইতিমধ্যে ঘটে যাওয়া কিছুর প্রতিক্রিয়া জানাতে হলে asynchronous event/queue ব্যবহার করুন।
- Service discovery-এর অস্তিত্ব আছে কারণ আধুনিক deployment-এ (autoscaling, containers, Kubernetes) instance address dynamic এবং স্বল্পস্থায়ী।
- Client-side discovery registry lookup এবং load-balancing logic caller-এর মধ্যে রাখে (যেমন, Netflix Eureka + Ribbon)।
- Server-side discovery সেই logic একটি load balancer বা platform feature-এর পেছনে লুকিয়ে রাখে (যেমন, Kubernetes Services)।
- একটি service mesh (Istio, Linkerd) sidecar proxy-এর মাধ্যমে discovery, load balancing, retries, এবং security সামলায়, application code-এর কাছে transparently।
- একটি service registry ক্রমাগত registration, health checking, এবং deregistration-এর মাধ্যমে কাজ করে যাতে শুধুমাত্র healthy address-গুলোই discoverable থাকে।
