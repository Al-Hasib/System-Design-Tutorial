# Module 2: Networking & Communication

প্রতিটা system design আলোচনা শেষ পর্যন্ত এসে দাঁড়ায় machine-গুলো আসলে কীভাবে একে অপরের সাথে কথা বলে তার উপর: কোন protocol byte বহন করে, client আর server-এর মাঝখানে কী থাকে, এবং অনেকগুলো server-এর মধ্যে কাজ নিরাপদে কীভাবে ছড়িয়ে দেওয়া হয়। এই module system design-এর communication layer নিয়ে কাজ করে — web-এর ভাষা হিসেবে HTTP ও REST, traffic-control layer হিসেবে load balancing ও proxy, আপনার backend-এর front door হিসেবে API gateway, এবং যখন সাধারণ request/response cycle যথেষ্ট দ্রুত নয় তখনকার জন্য real-time protocol। এই layer ভালোভাবে বুঝলে বাকি পুরো কোর্সে আপনি যে কোনো design-এর latency, availability, এবং scale নিয়ে reasoning করতে পারবেন।

## এই Module-এর Video সমূহ

| # | Title | Description | Link |
|---|-------|-------------|------|
| 06 | HTTP/HTTPS & REST APIs Explained | Web-এর request/response protocol কীভাবে কাজ করে, TLS কী যোগ করে, এবং REST কীভাবে resource-কে কেন্দ্র করে API structure করে। | [06-http-https-and-rest-apis](06-http-https-and-rest-apis/README.md) |
| 07 | Load Balancing Explained (Algorithms & L4 vs L7) | কীভাবে traffic একাধিক server-এ distribute হয়, common algorithm-গুলো, এবং transport-layer ও application-layer balancing-এর মধ্যে পার্থক্য। | [07-load-balancing-explained](07-load-balancing-explained/README.md) |
| 08 | Forward Proxy vs Reverse Proxy | একটা proxy আসলে কী করে, এবং কেন forward proxy client-কে সুরক্ষা দেয় আর reverse proxy server-কে সুরক্ষা ও ক্ষমতা দেয়। | [08-forward-proxy-vs-reverse-proxy](08-forward-proxy-vs-reverse-proxy/README.md) |
| 09 | API Gateway & Backend-for-Frontend Pattern | একটা single entry point কীভাবে microservices-এর জন্য auth, rate limiting, এবং routing handle করে, এবং কেন কিছু team প্রতিটা client type-এর জন্য আলাদা BFF যোগ করে। | [09-api-gateway-and-bff-pattern](09-api-gateway-and-bff-pattern/README.md) |
| 10 | WebSockets, Long Polling & Server-Sent Events | সাধারণ HTTP request/response যথেষ্ট দ্রুত না হলে real-time feature কীভাবে বানাবেন, এবং তিনটি প্রধান technique-এর মধ্যে কীভাবে বেছে নেবেন। | [10-websockets-long-polling-and-sse](10-websockets-long-polling-and-sse/README.md) |
