# এই বিষয়টি কেন গুরুত্বপূর্ণ: CDN (Content Delivery Network)

> **এক বাক্যে:** কোনো পরিমাণ backend optimization speed of light-কে হারাতে পারে না, তাই যদি আপনার user-রা আপনার server থেকে দূরে থাকে, একমাত্র প্রকৃত সমাধান হলো content-কে তাদের কাছাকাছি নিয়ে যাওয়া।

## এই ধারণার আগের পৃথিবী

আপনার server গুলো Virginia-তে। Sydney-র একজন user আপনার homepage load করছেন। Round trip প্রায় ২০০ ms — একটি কঠিন physical সীমা, কোনো tuning সমস্যা নয়। Page-টির ৬০টি resource প্রয়োজন: HTML, CSS, JavaScript bundle, font, এবং image। connection reuse এবং parallelism থাকা সত্ত্বেও, কিছু render হওয়ার আগে এটা শুদ্ধ network waiting-এর কয়েক সেকেন্ড।

এবার গুণ করুন। ২ MB-র একটি page গুণ ১ কোটি request মানে ২০ TB egress, যার সবটাই আপনার origin থেকে, যার সবটাই origin rate-এ billed হয়, যার সবটাই আপনার dynamic API traffic-এর সাথে একই bandwidth-এর জন্য প্রতিযোগিতা করছে। এবং যখন একটি video viral হয়, সেই traffic একসাথে, একটি একক location-এ এসে পৌঁছায়।

## যেসব সমস্যা এটি সমাধান করে

### ১. Distance যা আপনি optimize করে সরাতে পারবেন না
**আপনি যা দেখেন:** আপনার app office-এ instant মনে হয় কিন্তু আপনার অর্ধেক user-এর জন্য ধীরগতির, আর profiling দেখায় backend দ্রুত।

**কেন এটা ঘটে:** Latency মূলত propagation delay দ্বারা নিয়ন্ত্রিত। Sydney থেকে Virginia প্রায় ১৬,০০০ কিমি; fiber-এ light সেটা এক দিকে প্রায় ৮০ ms-এ কভার করে, আর প্রকৃত routing এটাকে আরও খারাপ করে। কোনো server upgrade এই সংখ্যা স্পর্শ করে না।

**একটি CDN কীভাবে এটি সমাধান করে:** Content শত শত শহরের edge server-এ cache করা থাকে। Sydney-র user-কে Sydney থেকেই serve করা হয় — ২০০+ ms-এর বদলে single-digit মিলিসেকেন্ড। এটাই একমাত্র mechanism যা আসলেই distance কমায়।

### ২. Origin bandwidth এবং cost
**আপনি যা দেখেন:** বিশাল egress bill এবং একটি origin যা কোনো মূল্যবান কাজের বদলে static file serving-এ saturated।

**কেন এটা ঘটে:** প্রতিটি image, script, এবং video-র প্রতিটি byte একটি জায়গা থেকে, বারবার, একইভাবে serve করা হয়।

**একটি CDN কীভাবে এটি সমাধান করে:** Edge ৯০-৯৯% traffic শুষে নেয়। আপনার origin প্রতিটি asset প্রতি edge location-এ প্রতি cache period-এ একবার serve করে। Egress cost উল্লেখযোগ্যভাবে কমে যায় (বেশি পরিমাণে CDN bandwidth সস্তা) এবং আপনার server তাদের CPU সেই request-এর জন্য ব্যয় করে যেগুলোর আসলেই তাদের প্রয়োজন।

### ৩. Traffic spike যা origin-কে বিপর্যস্ত করে
**আপনি যা দেখেন:** একটি product launch, একটি viral post, বা একটি marketing email আসে আর site আপনার কখনো provision না করা load-এর নিচে down হয়ে যায়।

**কেন এটা ঘটে:** সমস্ত traffic একটি নির্দিষ্ট origin capacity-তে গিয়ে মেলে।

**একটি CDN কীভাবে এটি সমাধান করে:** Edge network-এর কাছে আপনার কখনো provision করা capacity-র চেয়ে বহুগুণ বেশি aggregate capacity থাকে, এবং cacheable content-এর জন্য একটি spike সেখানেই শুষে নেওয়া হয়। এটাই DDoS protection-এরও প্রথম layer — volumetric attack এমন একটি network-এ আঘাত করে যা সেগুলো শুষে নেওয়ার জন্যই তৈরি, আপনার origin-এ নয়।

### ৪. Video এবং বড় file যা প্রচলিতভাবে serve করা যায় না
**আপনি যা দেখেন:** আপনার origin থেকে streaming video ক্রমাগত buffer করে এবং প্রচুর খরচ হয়।

**কেন এটা ঘটে:** Video বিশাল, bandwidth-intensive, এবং startup time-এর জন্য অত্যন্ত latency-sensitive।

**একটি CDN কীভাবে এটি সমাধান করে:** Video segment-এর edge caching-ই হলো ঠিক যা streaming-কে সম্ভব করে তোলে। Netflix এবং YouTube, architecturally, একটি catalog-যুক্ত CDN কোম্পানি — এটা তাদের জন্য কোনো optimization নয়, এটাই মূল design।

## যে মূল্য আপনাকে দিতে হয়

- **শত শত node জুড়ে Invalidation।** আপনি এক বছরের cache header সহ একটি খারাপ CSS file push করেছেন; এটা এখন ৩০০টি শহরে cache করা আছে। Purge আছে কিন্তু অসমভাবে propagate হয়। মানক প্রশমন হলো content-hashed filename (`app.a3f9c2.js`) যাতে নতুন content নতুন একটি URL পায় এবং পুরনো content শুধু request হওয়া বন্ধ হয়ে যায় — এটা জানা গুরুত্বপূর্ণ কারণ এটাই সমাধান, কোনো workaround নয়।
- **Stale content বিভ্রান্তি।** ভিন্ন মহাদেশের user-রা deploy-এর পর কিছু সময়ের জন্য আপনার site-এর ভিন্ন version দেখতে পারেন।
- **Dynamic এবং personalized content সহজে cache হয় না।** যেকোনো user-specific জিনিসকে হয় cache bypass করতে হবে অথবা সতর্কতার সাথে cache key ব্যবহার করতে হবে — এবং একটি cache-key ভুল যা একজন user-এর personalized page আরেকজনকে serve করে, তা শুধু একটি bug নয়, একটি গুরুতর data leak।
- **Cost এবং complexity।** আরেকটি vendor, আরেকটি config surface, request path-এ আরেকটি জিনিস যা misbehave করতে পারে। CDN misconfiguration প্রকৃত outage ঘটায়।
- **Debugging।** এমন একটি সমস্যা reproduce করা যা শুধুমাত্র একটি edge location-এ, একটি cached variant-এ ঘটে, তা সত্যিই কঠিন।

## কখন আপনার এটি প্রয়োজন — এবং কখন নয়

| যখন CDN ব্যবহার করবেন | যখন এড়িয়ে যাবেন |
|---|---|
| আপনি static asset (JS, CSS, image, font, video) serve করেন | এটা একটি internal tool যার user একই office-এ |
| আপনার user-রা ভৌগোলিকভাবে বিস্তৃত | সব traffic dynamic এবং uncacheable |
| Bandwidth cost উল্লেখযোগ্য | Traffic এতটাই কম যে origin serving সহজেই সস্তা |
| আপনার edge-এ DDoS absorption প্রয়োজন | — |
| আপনি বড় পরিসরে media serve করছেন | — |

## এটি কেন Interview-এ দেখা যায়

Image, video, বা global user base জড়িত যেকোনো design-এ একটি CDN থাকা উচিত, এবং YouTube বা Netflix design-এ একটি বাদ দেওয়া একটি উল্লেখযোগ্য ভুল। Diagram-এ এটি বসানোর বাইরে, interviewer-রা দেখেন: কী cacheable বনাম কী অবশ্যই origin-এ hit করতে হবে, আপনি কীভাবে invalidate করেন (এবং content-hashing trick), আপনি কী cache header set করেন, এবং আপনি বোঝেন কিনা যে dynamic personalized content-এর ভিন্ন handling প্রয়োজন। Video design-এ, বুঝতে পারা যে CDN হলো architecture-এর মূল অংশ — কোনো add-on নয় — এটাই মূল insight।

## এটি কীভাবে সংযুক্ত

একটি CDN হলো **caching** (topic ১৭) network edge-এ প্রয়োগ করা, এবং প্রতিটি edge node কার্যকরভাবে একটি ভৌগোলিকভাবে distributed **reverse proxy**। এটা **multi-region architecture**-এর সবচেয়ে সস্তা রূপ — static content-এর জন্য global infrastructure না চালিয়েই global presence। কাজ করার জন্য এটি **HTTP** cache-control semantics-এর উপর নির্ভর করে, এবং এটা **video streaming** এবং **file storage** case study-র একটি কেন্দ্রীয় উপাদান।

**পরবর্তী:** [Distributed Caching with Redis & Memcached](../19-distributed-caching-redis-and-memcached/why.md) — আপনার নিজের infrastructure-এর ভেতরের caching layer।
