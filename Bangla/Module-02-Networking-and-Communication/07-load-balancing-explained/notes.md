# Study Notes: Load Balancing

## Definitions

- **Load balancer:** একটা component যা server-এর একটা pool-এর সামনে বসে থাকে এবং একটা algorithm-এর ভিত্তিতে আগত traffic তাদের মধ্যে বণ্টন করে।
- **Health check:** একটা periodic probe (যেমন, `/health`-এ HTTP GET) যা load balancer একটা backend server উপলব্ধ কিনা তা নির্ধারণ করতে ব্যবহার করে।
- **Sticky session:** একটা নির্দিষ্ট client-এর সব request একই backend server-এ route করা, সাধারণত in-memory session state সংরক্ষণ করার জন্য।
- **Single point of failure (SPOF):** এমন একটা component যার failure পুরো system-কে নামিয়ে দেয়; load balancer-কে redundant বানাতে হবে যাতে এটা একটা SPOF না হয়ে যায়।

## Load Balancing Algorithms

| Algorithm | কীভাবে কাজ করে | সবচেয়ে উপযুক্ত | দুর্বলতা |
|-----------|---------------|----------|----------|
| Round robin | Request গুলো নির্দিষ্ট ক্রমে server-দের মধ্যে ঘোরে | Homogeneous server, uniform request cost | প্রকৃত server load উপেক্ষা করে |
| Weighted round robin | Round robin, কিন্তু বেশি-capacity-এর server বেশি turn পায় | ভিন্নধর্মী server capacity | Static weight real-time load-এর সাথে খাপ খায় না |
| Least connections | নতুন request সবচেয়ে কম active connection-ওয়ালা server-এ যায় | পরিবর্তনশীল request cost/duration | State track করতে সামান্য বেশি overhead |
| Weighted least connections | Least connections, server capacity দিয়ে weighted | ভিন্নধর্মী server + পরিবর্তনশীল request cost | Tune করা বেশি জটিল |
| IP hash | Client IP-এর hash server নির্ধারণ করে | Cookie ছাড়া session stickiness দরকার | Client IP skewed হলে অসম load; server পরিবর্তনে reshuffling |
| Least response time | Connection count এবং observed latency একত্রিত করে | Latency-sensitive service | পরিমাপ করতে বেশি overhead |
| Consistent hashing | Request/key কে একটা hash ring-এ server-এ map করে, পরিবর্তনে ন্যূনতম remapping | Caching layer, sharded backend | Implement করা বেশি জটিল (Module 6-এ কভার করা হয়েছে) |

## Layer 4 vs Layer 7 Load Balancing

| দিক | Layer 4 (Transport) | Layer 7 (Application) |
|--------|----------------------|--------------------------|
| OSI Layer | Transport (TCP/UDP) | Application (HTTP/HTTPS/gRPC, ইত্যাদি) |
| দৃশ্যমানতা | শুধু IP address + port | সম্পূর্ণ request: URL, header, cookie, body |
| Routing বুদ্ধিমত্তা | Content দিয়ে route করতে পারে না | Path, header, cookie, ইত্যাদি দিয়ে route করতে পারে |
| Performance | খুবই দ্রুত, low CPU overhead | বেশি CPU overhead (প্রতিটা request parse করে) |
| TLS termination | সাধারণত pass through হয় | Backend-দের পক্ষে TLS terminate করতে পারে |
| উদাহরণ প্রযুক্তি | AWS Network Load Balancer, IPVS | NGINX, HAProxy (L7 mode), AWS ALB, Envoy |
| ব্যবহারের ক্ষেত্র | High-throughput, non-HTTP protocol, extreme low latency | বেশিরভাগ আধুনিক web/API traffic, microservice routing |

সাধারণ নিয়ম: **L4 route করে connection। L7 route করে request।**

## Health Checks

- Active check: load balancer সময়সূচী অনুযায়ী সক্রিয়ভাবে একটা health endpoint-এ ping করে।
- Passive check: load balancer প্রকৃত traffic পর্যবেক্ষণ করে (যেমন, error rate, timeout) health অনুমান করতে।
- Unhealthy server গুলো rotation থেকে সরানো হয় যতক্ষণ না তারা আবার check pass করে — এটা self-healing traffic routing সক্ষম করে।

## Avoiding SPOF in the Load Balancing Layer

- একাধিক load balancer instance চালান।
- LB instance জুড়ে DNS round robin ব্যবহার করুন, বা VRRP/keepalived সহ একটা floating/virtual IP।
- একটা managed, multi-AZ load balancer service (যেমন, cloud provider-এর ALB/NLB) পছন্দ করুন যা design-এর দিক থেকে redundant।

## Summary

- Load balancing হলো সেই জিনিস যা horizontal scaling-কে বাস্তবে আসলেই কাজ করায়।
- Algorithm বাছাই নির্ভর করে server homogeneity, request cost variability, এবং session প্রয়োজনীয়তার উপর।
- L4 বনাম L7 হলো raw speed এবং routing বুদ্ধিমত্তার মধ্যে একটা tradeoff — বেশিরভাগ আধুনিক system HTTP traffic-এর জন্য L7 পছন্দ করে।
- Health check + redundant load balancer হলো সেই জিনিস যা failure-গুলোকে end user-দের কাছে অদৃশ্য করে তোলে।
