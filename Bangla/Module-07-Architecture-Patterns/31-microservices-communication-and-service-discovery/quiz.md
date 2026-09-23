# অনুশীলন ও Interview প্রশ্ন

**১. একটি service কখন অন্য একটি service-কে synchronously কল করবে আর কখন asynchronously?**
Synchronous call ব্যবহার করুন যখন caller-এর বর্তমান operation চালিয়ে যাওয়ার জন্য তাৎক্ষণিক উত্তর দরকার (যেমন, একটা order নিশ্চিত করার আগে real-time inventory চেক করা)। Asynchronous event ব্যবহার করুন যখন receiver শুধু ইতিমধ্যে ঘটে যাওয়া কোনো কিছুর প্রতিক্রিয়া জানাচ্ছে এবং caller-এর block হওয়ার দরকার নেই (যেমন, একটা confirmation email পাঠানো, analytics আপডেট করা)।

**২. Microservices কেন একে অপরের জন্য শুধু hardcoded IP address ব্যবহার করতে পারে না?**
আধুনিক deployment-এ, instance dynamically স্কেল আপ/ডাউন হয়, failure-এ replace হয়, এবং প্রতি restart-এ নতুন IP পায় (বিশেষত Kubernetes-এর মতো container orchestrator-এ)। একটা hardcoded address প্রায় সঙ্গে সঙ্গেই stale হয়ে যাবে, তাই service-গুলোর runtime-এ current, healthy address lookup করার একটা উপায় দরকার — service discovery।

**৩. Client-side এবং server-side service discovery-এর মধ্যে তুলনা করুন।**
Client-side discovery-তে, কল করা service নিজেই service registry query করে এবং কোন healthy instance কল করবে তা বেছে নেয় (যেমন, Netflix Eureka + Ribbon)। Server-side discovery-তে, client একটা stable endpoint (একটা load balancer বা platform abstraction) কল করে, আর সেই intermediary registry query করে এবং request forward করে (যেমন, Kubernetes Services)। Client-side caller-কে বেশি নিয়ন্ত্রণ দেয়; server-side একটা অতিরিক্ত hop-এর বিনিময়ে client code সহজ রাখে।

**৪. Service mesh কী, এবং basic service discovery-এর বাইরে এটা কোন সমস্যা সমাধান করে?**
একটি service mesh (যেমন, Istio, Linkerd) প্রতিটি service instance-এর পাশে একটা sidecar proxy deploy করে discovery, load balancing, retries/timeouts, mutual TLS encryption, এবং observability transparently সামলাতে — সবই application code-এর বাইরে। এটা প্রতিটি service/language-এ এই cross-cutting concern-গুলো পুনরায় implement করার সমস্যা সমাধান করে সেগুলোকে infrastructure-এ কেন্দ্রীভূত করার মাধ্যমে।

**৫. একটি service registry-তে একটা instance-এর lifecycle বর্ণনা করুন।**
Startup-এ, instance নিজেকে তার address এবং একটা health check endpoint দিয়ে register করে। Registry পর্যায়ক্রমে সেই health check poll করে; healthy instance তালিকাভুক্ত থাকে এবং caller-দের ফেরত দেওয়া হয়, unhealthy instance বাদ পড়ে। Graceful shutdown-এ instance নিজেকে deregister করে; crash হলে, ব্যর্থ health check শেষ পর্যন্ত সেটা সরিয়ে দেয়।

**৬. একটি interview-এ, আপনি একটা e-commerce checkout flow ডিজাইন করছেন। কোন অংশগুলো synchronous এবং কোনগুলো asynchronous হওয়া উচিত?**
Synchronous: payment authorization যাচাই করা এবং inventory availability নিশ্চিত করা, কারণ user purchase সম্পন্ন করার জন্য একটা নির্দিষ্ট হ্যাঁ/না-এর জন্য অপেক্ষা করছে। Asynchronous: order confirmation email পাঠানো, recommendation/analytics system আপডেট করা, এবং fulfillment-এর জন্য warehouse-কে জানানো — এগুলোর কোনোটারই checkout response block করার দরকার নেই।

**৭. Synchronous call-এর তুলনায় asynchronous, event-driven communication-এর প্রধান অসুবিধা কী?**
এটা eventual consistency নিয়ে আসে — অন্য service-গুলো একটা পরিবর্তন সাথে সাথে প্রতিফলিত নাও করতে পারে, এবং যেকোনো মুহূর্তে সামগ্রিক system state নিয়ে চিন্তা করা কঠিন হয়ে ওঠে। এটা infrastructure জটিলতাও যোগ করে (message broker, ordering guarantee, dead-letter handling)।

**৮. Service discovery ছাড়া শুধু একটা load balancer (Module 2-এ কভার করা হয়েছে) microservices-এর জন্য কেন যথেষ্ট নয়?**
একটা traditional load balancer-এর সাধারণত তার backend pool configure করা দরকার, যা static বা ধীরে আপডেট হয়। Service discovery সেই pool-কে dynamic এবং real time-এ self-updating বানায় যখন instance-গুলো register, deregister হয়, এবং health check pass/fail করে — server-side discovery মূলত একটা live service registry-এর সাথে সরাসরি তারযুক্ত একটা load balancer।

**৯. Netflix Eureka কী এবং এটা কোন discovery pattern প্রতিনিধিত্ব করে?**
Eureka হলো Netflix-এর open-source service registry, ঐতিহাসিকভাবে Ribbon client-side load balancer-এর সাথে জোড়া লাগানো। এটা client-side discovery pattern প্রতিনিধিত্ব করে: service-গুলো সরাসরি Eureka-কে healthy instance-এর একটা তালিকার জন্য query করে এবং নিজেরাই একটা বেছে নেয়।

**১০. যদি একটা downstream service ধীর হয়ে যায় (পুরোপুরি down না), এটা synchronous service-to-service call-এর সাথে কীভাবে interact করে, এবং Module 6-এর কোন pattern সাহায্য করে?**
একটা ধীর downstream service caller-কে block করতে পারে, resource (threads/connections) আটকে রাখতে পারে, এবং upstream-এ ধীরগতি cascade করতে পারে। Circuit breaker, timeout, এবং bulkhead (Module 6) দ্রুত fail করে, wait time-এ সীমা টেনে, এবং resource pool আলাদা করে সাহায্য করে যাতে একটা ধীর dependency পুরো কল করা service-কে ধ্বংস না করে।

**১১. Service discovery-এর পাশাপাশি একটা API gateway (Module 2) কী ভূমিকা পালন করে?**
API gateway সাধারণত বাহ্যিক-মুখী entry point যা client request-কে সঠিক internal service-এ route করে, প্রায়ই healthy backend instance খুঁজে পেতে internally server-side discovery ব্যবহার করে। এটা authentication, rate limiting, এবং request routing-এর মতো বিষয়গুলোও কেন্দ্রীভূত করে, যেখানে service discovery বিশেষভাবে প্রতিটি internal service-এর live instance খুঁজে পাওয়া সামলায়।

**১২. একটা team কেন একটা সম্পূর্ণ service mesh গ্রহণ করার বদলে Kubernetes-এর built-in service discovery বেছে নিতে পারে?**
Kubernetes Services + DNS + kube-proxy ইতিমধ্যে ন্যূনতম অতিরিক্ত operational বোঝা ছাড়াই out of the box শক্তিশালী server-side discovery এবং load balancing প্রদান করে। একটা সম্পূর্ণ service mesh উল্লেখযোগ্য জটিলতা যোগ করে (control plane, sidecar injection, certificate management) যা তখনই সার্থক হয় যখন আপনার এর অতিরিক্ত ফিচারগুলো দরকার — সূক্ষ্ম traffic control, সর্বত্র mutual TLS, সমৃদ্ধ observability — অনেক সংখ্যক service জুড়ে।
