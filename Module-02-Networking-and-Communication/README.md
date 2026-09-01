# Module 2: Networking & Communication

Every system design conversation eventually comes down to how machines actually talk to each other: what protocol carries the bytes, what sits between the client and the server, and how work gets spread across many servers safely. This module covers the communication layer of system design — HTTP and REST as the language of the web, load balancing and proxies as the traffic-control layer, API gateways as the front door to your backend, and real-time protocols for when a plain request/response cycle isn't fast enough. Understanding this layer well is what lets you reason about latency, availability, and scale in every design you'll do for the rest of this course.

## Videos in This Module

| # | Title | Description | Link |
|---|-------|-------------|------|
| 06 | HTTP/HTTPS & REST APIs Explained | How the web's request/response protocol works, what TLS adds, and how REST structures APIs around resources. | [06-http-https-and-rest-apis](06-http-https-and-rest-apis/README.md) |
| 07 | Load Balancing Explained (Algorithms & L4 vs L7) | How traffic gets distributed across servers, the common algorithms, and the difference between transport-layer and application-layer balancing. | [07-load-balancing-explained](07-load-balancing-explained/README.md) |
| 08 | Forward Proxy vs Reverse Proxy | What a proxy actually does, and why forward proxies protect clients while reverse proxies protect and empower servers. | [08-forward-proxy-vs-reverse-proxy](08-forward-proxy-vs-reverse-proxy/README.md) |
| 09 | API Gateway & Backend-for-Frontend Pattern | How a single entry point handles auth, rate limiting, and routing for microservices, and why some teams add a BFF per client type. | [09-api-gateway-and-bff-pattern](09-api-gateway-and-bff-pattern/README.md) |
| 10 | WebSockets, Long Polling & Server-Sent Events | How to build real-time features when plain HTTP request/response is too slow, and how to choose between the three main techniques. | [10-websockets-long-polling-and-sse](10-websockets-long-polling-and-sse/README.md) |
