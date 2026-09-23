# Notes: Microservices Communication ও Service Discovery

## সংজ্ঞা

- **Synchronous communication**: Request/response call (REST, gRPC) যেখানে caller reply না পাওয়া পর্যন্ত block হয়ে থাকে।
- **Asynchronous communication**: Event/message-based communication (Kafka বা RabbitMQ-এর মতো একটা broker/queue-এর মাধ্যমে) যেখানে receiver message process করার জন্য caller block হয়ে থাকে না।
- **Service discovery**: এমন একটি mechanism যা একটি service-কে runtime-এ, কোনো hardcoded address ছাড়াই, অন্য একটি service-এর healthy instance-এর network location খুঁজে পেতে দেয়।
- **Service registry**: বর্তমানে উপলব্ধ service instance এবং তাদের health status-এর একটি database (যেমন, Consul, Eureka, etcd, Kubernetes API server)।
- **Service mesh**: একটি infrastructure layer (যেমন, Istio, Linkerd) যা per-instance sidecar proxy (প্রায়ই Envoy) ব্যবহার করে discovery, load balancing, retries, mTLS, এবং observability transparently সামলায়।
- **Sidecar proxy**: প্রতিটি service instance-এর পাশে deploy করা একটি proxy process যা তার সমস্ত network traffic intercept করে।

## Communication Style তুলনা

| দিক | Synchronous (REST/gRPC) | Asynchronous (Queue/Event) |
| ---------------- | ----------------------------------------------- | ------------------------------------------------------ |
| Caller block হয়? | হ্যাঁ, response-এর জন্য অপেক্ষা করে | না, fire-and-forget বা event publish |
| Coupling | আরও শক্ত — callee-এর availability caller-কে প্রভাবিত করে | আরও ঢিলা — সময়ের দিক থেকে decoupled |
| Use case | তাৎক্ষণিক উত্তর দরকার (যেমন, stock চেক করা) | ঘটে যাওয়া কোনো কিছুর প্রতিক্রিয়া জানানো (যেমন, email পাঠানো) |
| Failure handling | timeouts/retries/circuit breaker দরকার | Broker message buffer করে; consumer পরে catch up করে |
| Consistency | তাৎক্ষণিক state নিয়ে চিন্তা করা সহজ | Service জুড়ে eventual consistency |

## Service Discovery Pattern তুলনা

| Pattern | Lookup কে করে | উদাহরণ প্রযুক্তি | সুবিধা | অসুবিধা |
| --------------------- | ------------------------------------------------------------------ | ------------------------------- | ----------------------------------------------------------- | --------------------------------------------------- |
| Client-side discovery | কল করা service registry query করে এবং একটা instance বেছে নেয় | Netflix Eureka + Ribbon | Load balancing-এর ওপর সম্পূর্ণ client control | প্রতিটি client/language-এ discovery logic ডুপ্লিকেট হয় |
| Server-side discovery | একটা load balancer/router client-এর হয়ে registry query করে | Kubernetes Services, AWS ELB | Client সহজ থাকে, কোনো discovery logic দরকার হয় না | অতিরিক্ত network hop; LB critical infra |
| Service mesh | Sidecar proxy transparently discovery সামলায় | Istio, Linkerd (Envoy sidecar) | Discovery, LB, retries, security, observability কেন্দ্রীভূত করে | mesh নিজেই চালানোর জন্য উচ্চ operational জটিলতা |

## Service Registry Lifecycle

1. **Register** — startup-এ নতুন instance নিজেকে (address + health check endpoint) registry-তে ঘোষণা করে।
2. **Health check** — registry পর্যায়ক্রমে instance-এর health endpoint poll করে।
3. **Serve lookups** — registry শুধু healthy instance address caller-দের ফেরত দেয়।
4. **Deregister** — graceful shutdown-এ instance নিজেকে সরিয়ে নেয়, অথবা crash-এ ব্যর্থ health check-এর পর down বলে চিহ্নিত হয়।

## Bullet সারসংক্ষেপ

- Caller-এর তাৎক্ষণিক উত্তর দরকার নাকি শুধু অতীতের কোনো event-এর প্রতিক্রিয়া জানাচ্ছে তার ভিত্তিতে sync বনাম async বেছে নিন।
- Service discovery autoscaled/containerized environment-এ dynamic, ephemeral instance address-এর সমস্যা সমাধান করে।
- তিনটি discovery pattern: client-side, server-side, এবং service-mesh (sidecar-based) — discovery/load-balancing logic কোথায় থাকে তাতে ভিন্ন।
- একটি service registry registration এবং health check-এর মাধ্যমে ক্রমাগত instance health track করে যাতে শুধু live instance-ই ফেরত দেওয়া হয়।
- বাস্তব system দুটো communication style-ই মেশায় এবং প্রায়ই advanced traffic control-এর জন্য platform-native discovery-এর (যেমন, Kubernetes) ওপর একটা service mesh layer করে।
