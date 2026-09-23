# Study Notes: Forward Proxy vs Reverse Proxy

## Definitions

- **Proxy:** একটি intermediary server যা client এবং একটি destination server-এর মধ্যে request এবং response relay করে।
- **Forward proxy:** এমন একটি proxy যা client-দের সামনে বসে এবং বাইরের দুনিয়ার কাছে তাদের represent করে; destination server proxy-কে দেখে, প্রকৃত client-কে নয়।
- **Reverse proxy:** এমন একটি proxy যা server-দের সামনে বসে এবং বাইরের দুনিয়ার কাছে তাদের represent করে; client proxy-কে দেখে, প্রকৃত backend server-কে নয়।

## Forward Proxy vs Reverse Proxy Comparison

| Aspect | Forward Proxy | Reverse Proxy |
|--------|----------------|-----------------|
| কে এটি deploy/configure করে | Client-পক্ষ (যেমন, কোম্পানি তার কর্মীদের জন্য) | Server-পক্ষ (যেমন, কোম্পানি তার নিজস্ব backend-এর জন্য) |
| এটি কাকে লুকায় | Client-কে, server থেকে | Server-কে (এবং এর topology-কে), client থেকে |
| অন্য পক্ষ কী দেখে | Server শুধু proxy-র identity দেখে | Client শুধু proxy-র identity দেখে |
| সাধারণ ব্যবহার | Content filtering, anonymity, geo-restriction এড়ানো, client-side caching, outbound traffic monitor করা | TLS termination, load balancing, response caching, internal architecture লুকানো, compression, security (WAF) |
| উদাহরণ product | Corporate proxy server, Squid, VPN service | NGINX, HAProxy, Envoy, cloud load balancer (ALB), Cloudflare |
| Client awareness | Client নিজে থেকেই এটি ব্যবহার করার জন্য configure করা থাকে | Client-এর কোনো ধারণা থাকে না যে একটি proxy/backend fleet বিদ্যমান |

## Relationship to Load Balancing

- Layer 7-এ কাজ করা একটি load balancer, technically, একটি reverse proxy যার একটি feature হলো load-distribution algorithm।
- Reverse proxy হলো বিস্তৃত category; load balancing, TLS termination, caching, এবং compression হলো এর সম্পাদিত সাধারণ কাজ।
- প্রতিটি reverse proxy load-balance করে না (কিছু শুধু caching/security-র জন্য একটি একক backend-এ proxy করে), এবং প্রতিটি load balancer L7 নয় (L4 balancer একইভাবে application content-এর full proxying করে না)।

## Common Reverse Proxy Responsibilities

- TLS/SSL termination
- Static বা প্রায়শই request করা content caching করা
- Compression (gzip/br)
- Backend pool জুড়ে load balancing
- Internal network topology এবং প্রকৃত server IP লুকানো
- Request/response rewriting, header injection
- Basic security filtering (rate limiting, WAF rule)

## Common Forward Proxy Responsibilities

- Outbound traffic-এর জন্য content filtering / access control
- Destination server থেকে client-এর identity anonymize করা
- প্রায়শই request করা external resource caching করা
- Compliance-এর জন্য outbound traffic logging/monitoring
- Network/geographic restriction এড়ানো

## Summary

- মনে রাখার এক বাক্য: **forward proxy client-কে রক্ষা করে, reverse proxy server-কে রক্ষা করে।**
- দুটোই intermediary; পার্থক্যটা সম্পূর্ণভাবে নির্ভর করে কোন পক্ষ এগুলো deploy করে এবং এগুলো কার identity রক্ষা করে তার উপর।
- Load balancer সাধারণত reverse proxy-র একটি বিশেষায়িত ক্ষেত্র, আলাদা কোনো concept নয়।
