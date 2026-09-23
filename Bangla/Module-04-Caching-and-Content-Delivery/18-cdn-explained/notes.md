# Study Notes: CDN (Content Delivery Network) Explained

## সংজ্ঞাসমূহ (Definitions)

- **CDN (Content Delivery Network)**: server-এর একটি globally distributed system যা end user-দের physically কাছাকাছি অবস্থান থেকে content cache এবং delivery করে।
- **Origin server**: authoritative server যেখানে content-এর প্রকৃত, canonical copy থাকে।
- **Edge server**: end user-দের কাছাকাছি অবস্থিত একটি CDN server, যা origin content-এর copy cache করে।
- **PoP (Point of Presence)**: একটি physical data center location যেখানে একটি CDN এক বা একাধিক edge server পরিচালনা করে।
- **Cache-Control headers**: HTTP response header (যেমন, `Cache-Control: max-age=3600`) যা cache-কে (browser, CDN) বলে দেয় revalidate করার আগে content কতক্ষণ রাখতে হবে।
- **Purge/Invalidation**: TTL শেষ হওয়ার আগে edge server থেকে cached content সরিয়ে ফেলার একটি স্পষ্ট action।
- **Anycast**: একটি routing কৌশল যেখানে একই IP address একাধিক physical location থেকে announce করা হয়; network routing (BGP) topologically নিকটতম-টিতে traffic পাঠায়।
- **Edge computing**: শুধু origin-এ না রেখে CDN edge server-এই সরাসরি application logic চালানো।

## Static বনাম Dynamic Content Caching

| দিক | Static Content | Dynamic Content |
|---|---|---|
| উদাহরণ | Image, CSS, JS bundle, video, downloadable file, static HTML | Personalized page, API response, search result |
| Cacheability | সব user-এর জন্য একই response — cache করা সহজ | প্রতি user/request-এ ভিন্ন — নিরাপদভাবে cache করা কঠিন |
| সাধারণ TTL | মিনিট থেকে দিন (প্রায়ই দীর্ঘ, update-এর জন্য cache-busting filename সহ) | সেকেন্ড (micro-caching) অথবা মোটেও cache করা হয় না |
| আধুনিক কৌশল | Standard edge caching | Edge computing, micro-caching, header/cookie-ভিত্তিক cache key |

## Request Routing কৌশল

| কৌশল | এটি কীভাবে কাজ করে | সুবিধা | অসুবিধা |
|---|---|---|---|
| DNS-based routing | DNS resolver query-র ভৌগোলিক/network location অনুযায়ী ভিন্ন IP ফেরত দেয় | সহজ, ব্যাপকভাবে সমর্থিত | DNS caching/resolver location accuracy কমাতে পারে; ধীর failover |
| Anycast (BGP) | একই IP অনেক location থেকে announce করা হয়; internet routing "নিকটতম" announcer-এ traffic পাঠায় | দ্রুত, স্বয়ংক্রিয় failover, কোনো DNS trickery প্রয়োজন নেই | BGP-level infrastructure প্রয়োজন; পরিচালনা করা বেশি জটিল |

## CDN Invalidation পদ্ধতি

| পদ্ধতি | বিবরণ |
|---|---|
| TTL / Cache-Control headers | Origin নির্দিষ্ট করে দেয় edge server revalidate করার আগে কতক্ষণ একটি response cache করতে পারবে |
| Manual Purge | সব edge server জুড়ে অবিলম্বে একটি cached object evict করার জন্য API/dashboard action |
| Versioned/cache-busting URLs | Filename/URL পরিবর্তন করা (যেমন, একটি hash যোগ করা) যাতে একটি "নতুন" resource ফ্রেশভাবে fetch করা হয়, সম্পূর্ণভাবে invalidation এড়িয়ে |

## মূল সংখ্যা / Rule of Thumb

- Fiber-এ speed of light প্রায় ২,০০,০০০ km/s; প্রায় ১৫,০০০ কিমি জুড়ে একটি round trip (যেমন, US থেকে Singapore) একাই যেকোনো server processing-এর আগে প্রায় ১০০+ ms খরচ করে।
- CDN ভৌগোলিকভাবে দূরের user-দের জন্য time-to-first-byte শত শত মিলিসেকেন্ড থেকে কমিয়ে কয়েক দশ মিলিসেকেন্ডে নামিয়ে আনতে পারে, একটি কাছাকাছি edge থেকে serve করার মাধ্যমে।
- বড় CDN provider-রা পৃথিবীজুড়ে শত থেকে হাজার হাজার PoP পরিচালনা করে।
- ভালোভাবে cache করা static asset edge-এ ৯০% এর বেশি cache hit ratio অর্জন করতে পারে, যা origin load নাটকীয়ভাবে কমিয়ে দেয়।

## সারসংক্ষেপ (Summary)

- CDN network distance-এর কারণে সৃষ্ট latency কমানোর জন্য content-কে user-দের physically কাছাকাছি cache করে।
- Origin source of truth ধারণ করে; PoP-এর edge server cached copy ধারণ করে।
- Static content সহজ জয়; dynamic content নিরাপদে উপকৃত হতে edge computing বা micro-caching প্রয়োজন।
- নিকটতম edge server-এ routing DNS-based routing বা Anycast ব্যবহার করে।
- CDN cache-এরও অন্য যেকোনো cache-এর মতো invalidation strategy প্রয়োজন (TTL, purge, cache-busting URL)।
- CDN performance tool এবং infrastructure protection (traffic spike, DDoS absorption) উভয় হিসেবেই কাজ করে।
