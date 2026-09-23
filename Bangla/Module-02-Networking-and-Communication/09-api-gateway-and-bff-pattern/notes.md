# Study Notes: API Gateway ও Backend-for-Frontend

## Definitions

- **API Gateway:** একটি microservices আর্কিটেকচারের সামনে একটি একক entry point যা routing, authentication, rate limiting, transformation, aggregation, এবং observability-কে কেন্দ্রীভূত করে।
- **Backend-for-Frontend (BFF):** প্রতিটি client type-এর (mobile, web, partner) জন্য তৈরি একটি dedicated, পাতলা backend layer যা সেই client-এর চাহিদা অনুযায়ী underlying service গুলোর call aggregate ও shape করে।
- **Aggregation:** client-side round trip কমাতে একাধিক internal service call-কে client-এর জন্য একটি একক response-এ একত্রিত করা।
- **Cross-cutting concern:** অনেক service জুড়ে প্রয়োজনীয় functionality (auth, logging, rate limiting) যা প্রতিটি service-এ ডুপ্লিকেট করার চেয়ে কেন্দ্রীভূত করা ভালো।

## Reverse Proxy বনাম API Gateway বনাম BFF

| দিক | Reverse Proxy | API Gateway | BFF |
|--------|----------------|--------------|-----|
| প্রাথমিক উদ্দেশ্য | Backend server লুকানো/সুরক্ষা করা, basic routing/TLS | cross-cutting logic সহ microservices-এর জন্য unified entry point | Client-specific API shaping এবং aggregation |
| Business/API semantics সম্পর্কে সচেতনতা | কম (মূলত transport/HTTP level) | মাঝারি-উচ্চ (route, auth, প্রতি API rate limit) | উচ্চ (একটি client-এর ঠিক চাহিদা অনুযায়ী তৈরি) |
| Per-client customization | নেই | সাধারণত সব client জুড়ে অভিন্ন | প্রতিটি client type অনুযায়ী সম্পূর্ণ customized |
| সাধারণ মালিক | Infra/platform team | Platform/API team | সেই client-এর মালিক team (যেমন, mobile team) |
| উদাহরণ | NGINX, HAProxy | Kong, Amazon API Gateway, Apigee, Netflix Zuul | প্রতিটি client type-এর জন্য custom Node/Go/Java service |

## API Gateway-র মূল দায়িত্বসমূহ

- সঠিক backend service-এ request routing (path/header-ভিত্তিক)
- Authentication ও authorization (কেন্দ্রীভূত token/session validation)
- প্রতি client/API key rate limiting ও throttling
- Request/response transformation (protocol translation, payload পুনর্বিন্যাস)
- Aggregation (একাধিক service-এ fan-out, একটি response-এ একত্রিতকরণ)
- কেন্দ্রীভূত observability: logging, metrics, tracing

## BFF কেন বিদ্যমান

- বিভিন্ন client-এর ভিন্ন ভিন্ন সীমাবদ্ধতা আছে: mobile (bandwidth/battery-সীমিত, minimal aggregated payload চায়), web (সমৃদ্ধ payload, আরও granular call), partner API (সম্পূর্ণ ভিন্ন auth/data shape)।
- সব client-কে সেবা দেওয়ার চেষ্টা করা একটি single generic API সাধারণত optional field ও conditional logic দিয়ে ভারী হয়ে যায়।
- BFF client-specific logic-কে তার নিজস্ব layer-এ আলাদা করে, যার মালিকানা সেই client-এর দায়িত্বপ্রাপ্ত team-এর হাতে থাকে, যা autonomy বাড়ায় এবং coordination overhead কমায়।

## Tradeoffs / Failure Mode

- **Single point of failure:** Gateway সমস্ত traffic-এর path-এ বসে থাকে — অবশ্যই redundant ভাবে deploy করতে হবে (load balancer-এর মতো একই নীতি)।
- **বাড়তি latency:** প্রতিটি request একটি অতিরিক্ত hop এবং processing step দিয়ে যায়।
- **Gateway sprawl / "edge monolith" anti-pattern:** সময়ের সাথে সাথে business logic gateway-তে ঢুকে পড়া একে পরিবর্তনের জন্য একটি bottleneck এবং team জুড়ে coupling-এর একটি shared point বানিয়ে দেয়।
- **BFF duplication:** একাধিক BFF অনুরূপ aggregation logic ডুপ্লিকেট করতে পারে — অর্জিত autonomy-র জন্য এটি একটি গ্রহণযোগ্য tradeoff, তবে নজরে রাখার মতো।

## Summary

- API Gateway = microservices আর্কিটেকচারের জন্য একক বুদ্ধিমান front door; cross-cutting concern গুলোকে কেন্দ্রীভূত করে।
- BFF = প্রতিটি client type-এর জন্য একটি dedicated, tailored backend, যা প্রায়ই gateway-র পেছনে বা পাশাপাশি থাকে।
- উভয়ই reverse-proxy foundation-এর উপর তৈরি কিন্তু plain reverse proxy-তে না থাকা application/business সচেতনতা যোগ করে।
