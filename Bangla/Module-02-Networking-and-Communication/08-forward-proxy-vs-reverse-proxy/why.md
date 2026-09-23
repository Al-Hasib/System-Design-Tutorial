# Why This Topic Matters: Forward Proxy vs Reverse Proxy

> **এক বাক্যে:** দুটোই একটি connection-এর মাঝখানে বসে থাকে, কিন্তু একটি client-এর হয়ে কাজ করে এবং অন্যটি server-এর হয়ে — এবং এদের মধ্যে বিভ্রান্তি তৈরি হলে security control এমন জায়গায় বসানো হয় যেখানে সেগুলো কাউকেই রক্ষা করে না।

## The World Before This Idea

কোনো intermediary ছাড়া, প্রতিটি client সরাসরি প্রতিটি server-এর সাথে কথা বলে। শুনতে পরিষ্কার মনে হলেও, এটি বেশ কিছু গুরুত্বপূর্ণ জিনিস অসম্ভব করে তোলে:

- আপনার backend server-গুলোর IP address সবার জন্য প্রকাশ্য, তাই সেগুলো সরাসরি আক্রমণযোগ্য।
- প্রতিটি backend-কে নিজে থেকেই TLS terminate করতে হয়, নিজস্ব certificate manage করতে হয়, নিজস্ব rate limit প্রয়োগ করতে হয়, এবং নিজস্ব access log লিখতে হয়।
- একটি কোম্পানির কাছে তার কর্মীদের machine কী কী পর্যন্ত পৌঁছাতে পারবে সে বিষয়ে policy প্রয়োগ করার কোনো উপায় থাকে না।
- একটি shared cache রাখার কোনো জায়গা থাকে না, কারণ path-এ কোনো shared point নেই।

একটি proxy আসলে connection-এর উপর একটি ইচ্ছাকৃত choke point মাত্র — এবং একটি choke point হলো এমন জায়গা যেখানে আপনি কিছু *করতে* পারেন।

## The Problems It Solves

### ১. প্রতিটি service-এ Cross-cutting concern-এর পুনরাবৃত্তি
**যা আপনি দেখেন:** TLS termination, gzip compression, request logging, rate limiting, এবং IP allowlisting আলাদাভাবে বারোটি service-এ implement করা হয়েছে, প্রতিটিতে সামান্য ভিন্নভাবে, এবং তার মধ্যে তিনটি পুরনো হয়ে গেছে।

**কেন এটি ঘটে:** কোনো shared entry point না থাকায়, প্রতিটি service-কেই সবকিছু নিজে থেকে handle করতে হয়।

**একটি reverse proxy এটি কীভাবে সমাধান করে:** সবকিছুর সামনে একটি জায়গা TLS (একটি certificate lifecycle দিয়ে), compression, logging, এবং basic filtering handle করে। Service-গুলো তখন শুধু plain HTTP server হয়ে business logic করতে পারে। এটাই মূল কারণ যে Nginx, HAProxy, এবং Envoy প্রায় প্রতিটি production stack-এ থাকে।

### ২. Backend সরাসরি internet-এর সামনে উন্মুক্ত
**যা আপনি দেখেন:** একটি scan আপনার application server খুঁজে পায়, এবং সেগুলো সরাসরি bot, scraper, এবং exploit attempt থেকে traffic পাচ্ছে।

**কেন এটি ঘটে:** client যদি সরাসরি backend-এর সাথে connect করে, তাহলে backend-কে publicly routable হতে হয়।

**একটি reverse proxy এটি কীভাবে সমাধান করে:** শুধুমাত্র proxy public থাকে। Backend private network-এ থাকে এবং শুধু সেখান থেকেই connection গ্রহণ করে। এটি attack surface-কে একটি hardened component-এ সীমাবদ্ধ করে এবং WAF ও DDoS filtering-এর জন্য একটি স্বাভাবিক জায়গা দেয়।

### ৩. প্রতিটি request application-এ পৌঁছায়, এমনকি একই রকম request-ও
**যা আপনি দেখেন:** একই logo, একই CSS bundle, এবং একই জনপ্রিয় API response প্রতি সেকেন্ডে হাজার হাজার বার application দ্বারা compute ও serve করা হচ্ছে।

**কেন এটি ঘটে:** কোনো shared intermediary না থাকায়, একটি shared cached copy রাখার কোনো জায়গা থাকে না।

**একটি proxy এটি কীভাবে সমাধান করে:** একটি reverse proxy response cache করতে পারে এবং application স্পর্শ না করেই সেগুলো serve করতে পারে। (একটি forward proxy এর ঠিক আয়না-প্রতিরূপ কাজ করে — অনেক client-এর হয়ে caching করে যাতে একটি জনপ্রিয় resource পুরো office-এর জন্য একবারই fetch করা হয়।)

### ৪. Network থেকে কী বের হচ্ছে তার উপর কোনো নিয়ন্ত্রণ নেই
**যা আপনি দেখেন:** একটি compromise হওয়া internal service একটি বাহ্যিক host-এ data exfiltrate করা শুরু করে, এবং কেউ তা লক্ষ্যও করে না বা থামায়ও না।

**কেন এটি ঘটে:** Outbound traffic নিয়ন্ত্রণহীন।

**একটি forward proxy এটি কীভাবে সমাধান করে:** সব outbound request একটি proxy-র মধ্য দিয়ে চালিত হয় যা destination allowlist করতে পারে, প্রতিটি request log করতে পারে, এবং বাকি সব block করতে পারে। এভাবেই corporate content filtering, egress auditing, এবং স্থিতিশীল outbound IP address (যখন কোনো partner *আপনাকে* allowlist করে তখন প্রয়োজন হয়) বাস্তবায়ন করা হয়।

## The Price You Pay

- **আরেকটি hop, আরেকটি failure domain।** proxy latency যোগ করে এবং নিজেই এমন কিছু হয়ে ওঠে যাকে redundant ও monitor করতে হয়।
- **Debugging কঠিন হয়ে যায়।** `X-Forwarded-For` সঠিকভাবে handle না করলে client IP প্রতিস্থাপিত হয়ে যায় — এবং এটি ভুলভাবে handle করা broken rate limiting, ভুল geolocation, এবং IP-spoofing vulnerability-র একটি ক্লাসিক উৎস।
- **Caching bug ভয়ংকর হতে পারে।** একটি ভুল configured cache key একজন user-এর personalized response আরেকজনকে serve করে ফেলতে পারে। একটি shared layer-এ caching করার জন্য auth-নির্ভর response নিয়ে সতর্কতা প্রয়োজন।
- **Configuration drift।** Proxy config নিজেই একটি codebase হয়ে ওঠে, এবং একটি খারাপ reload একবারেই সবকিছু বন্ধ করে দিতে পারে।

## When You Need It — and When You Don't

| Reverse proxy যখন | Forward proxy যখন |
|---|---|
| আপনি server operate করছেন এবং একটি একক public entry point চান | আপনি client operate করছেন এবং outbound traffic-এর উপর policy দরকার |
| আপনি centralized TLS, caching, compression, logging চান | Partner allowlisting-এর জন্য আপনার একটি স্থিতিশীল egress IP দরকার |
| আপনি backend লুকাতে ও রক্ষা করতে চান | Network থেকে কী বের হচ্ছে তার auditing বা filtering দরকার |
| আপনার service-জুড়ে request routing দরকার | অনেক client-এর জন্য একটি shared cache চান |

## Why This Shows Up in Interviews

Interviewer-রা এটি জিজ্ঞেস করে conceptual precision যাচাই করার জন্য, কারণ এই দুটি উপর থেকে দেখতে একই রকম এবং candidate-রা প্রায়ই এদের গুলিয়ে ফেলে। পরিষ্কার framing — *একটি forward proxy client-পক্ষ দ্বারা deploy করা হয় এবং internet-এর কাছে client-দের represent করে; একটি reverse proxy server-পক্ষ দ্বারা deploy করা হয় এবং client-দের কাছে server-দের represent করে* — এক বাক্যেই এর উত্তর দেয়। Follow-up প্রশ্নগুলো সাধারণত যায় একটি reverse proxy কীসের জন্য ভালো এবং এটি একটি load balancer ও API gateway থেকে কীভাবে আলাদা তার দিকে।

## How It Connects

Reverse proxy হলো সেই সাধারণ category যার বিশেষায়িত রূপ হলো **load balancer**, **API gateway**, এবং **CDN edge node**। এটি হলো **TLS termination**-এর (security topic দেখুন), **HTTP caching layer**-এর, এবং **rate limiting**-এর enforcement point-এর স্বাভাবিক ঘর। এই category-টা বুঝে ফেললে সেগুলো সব আলাদা আবিষ্কার না মনে হয়ে বরং একই জিনিসের ভিন্ন রূপ মনে হবে।

**পরবর্তী:** [API Gateway & Backend-for-Frontend Pattern](../09-api-gateway-and-bff-pattern/why.md) — সেই entry point যখন প্রকৃত intelligence অর্জন করে তখন কী ঘটে।
