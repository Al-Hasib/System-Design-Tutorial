# এই টপিকটি কেন গুরুত্বপূর্ণ: Zero-Downtime Deployments & Database Migrations

> **এক বাক্যে:** প্রতিটি deploy এমন একটা মুহূর্ত যখন আপনার code-এর দুটো version একসাথে অস্তিত্বে থাকে, আর আপনার schema পরিবর্তন যদি ধরে নেয় যে শুধু একটা version চলছে, তাহলে আপনি নিজেই একটা outage schedule করে ফেলেছেন।

## এই ধারণার আগের পৃথিবী

**Maintenance window।** Deploy হতো রবিবার রাত ২টায়, "we'll be back soon" পেজের আড়ালে। যেহেতু deploy করা কষ্টকর, তাই সেগুলো বিরল হতো; যেহেতু সেগুলো বিরল, তাই প্রতিটিতে এক মাসের পরিবর্তন একসাথে জড়ো হতো; যেহেতু প্রতিটি বড়, তাই failure ঘন ঘন হতো এবং diagnose করা কঠিন হতো। Deploy করার খরচই deploy-কে আরও খারাপ করে তুলত, যা আবার খরচ আরও বাড়িয়ে দিত। এই চক্রে আটকে থাকা team-গুলো ধীরে ধীরে ship করত এবং নিজেদের release process-কেই ভয় পেত।

**যে migration site-টাকে ফেলে দিয়েছিল।** একজন developer এক migration-এ `user_name`-এর নাম বদলে `username` করলেন। এটা চালু হলো। পুরনো code-এর প্রতিটি instance — যেগুলো rolling update-এর সময়েও তখনও traffic serve করছিল — সাথে সাথেই error ছুড়তে শুরু করল কারণ তারা যে column query করে সেটা আর অস্তিত্বেই নেই। Deploy-টা rollback করা হলো, কিন্তু migration-টা reversible ছিল না, আর এখন *নতুন* code-ও চলতে পারছে না। Site দুই দিক থেকেই বন্ধ।

## এটি যেসব সমস্যার সমাধান করে

### ১. Shipping-এর খরচ হিসেবে Downtime
**আপনি যা দেখেন:** Scheduled outage, কম-traffic সময়েই সীমাবদ্ধ release, এবং একটা release process যাতে রাতে বেশ কয়েকজন মানুষকে জেগে থাকতে হয়।

**এটা কেন ঘটে:** সব instance একসাথে replace করার মানে হলো এমন একটা window যেখানে কিছুই serve করছে না।

**Rolling deploy কীভাবে এটা সমাধান করে:** একটা load balancer-এর পেছনে একবারে অল্প কয়েকটা instance replace করো। একটাকে rotation থেকে বের করো, তার চলমান request drain করো, সেটাকে update করো, health-check করে আবার ফিরিয়ে আনো, পুনরাবৃত্তি করো। Capacity সামান্য কমে; availability কমে না। এটাই deploy করাকে একটা event না বানিয়ে একটা স্বাভাবিক দিনের কাজ বানিয়ে দেয়।

### ২. খারাপ release একসাথে সবার কাছে পৌঁছানো
**আপনি যা দেখেন:** একটা সূক্ষ্ম bug — memory leak, ধীরগতির query, বা একটা ভাঙা edge case — যা testing-এ ধরা পড়েনি, এখন ১০০% ব্যবহারকারীকে প্রভাবিত করছে।

**এটা কেন ঘটে:** পুরো exposure হওয়ার আগে production-এ কোনো exposure ছিল না।

**Canary এবং blue-green কীভাবে এটা সমাধান করে:** একটা canary নতুন version-এ traffic-এর একটা ছোট percentage পাঠায় এবং পুরনো version-এর সাথে error rate ও latency তুলনা করে। ১০০% ব্যবহারকারীর বদলে ১% ব্যবহারকারীর মধ্যেই সমস্যা ধরা পড়ে। Blue-green পুরনো environment-এর পাশাপাশি সম্পূর্ণ নতুন environment চালায় এবং একবারে traffic সুইচ করে — যা double capacity সাময়িকভাবে চালানোর খরচে প্রায়-তাৎক্ষণিক rollback দেয়, শুধু আবার সুইচ করে ফিরে গেলেই। দুটোই একই জিনিস দেয়: একটা সীমাবদ্ধ blast radius এবং বের হওয়ার একটা দ্রুত পথ।

### ৩. Schema পরিবর্তন যা code-এর একটা মাত্র version ধরে নেয়
**আপনি যা দেখেন:** উপরের rename পরিস্থিতিটা। অথবা কোনো default ছাড়া একটা `NOT NULL` column যোগ করা, যাতে পুরনো code-এর insert ব্যর্থ হয়। অথবা এমন একটা column drop করা যা আগের version-ও তখনও select করে।

**এটা কেন ঘটে:** যেকোনো rolling deploy-এর সময়, পুরনো ও নতুন code **একই database-এ একই সাথে** চলে। শুধু নতুন code-এর সাথে compatible একটা migration সাথে সাথে পুরনো code-কে ভেঙে ফেলে, আর শুধু পুরনো code-এর সাথে compatible একটা migration rollback-কে অসম্ভব করে তোলে।

**Expand-and-contract কীভাবে এটা সমাধান করে:** প্রতিটি breaking change-কে আলাদা আলাদাভাবে deploy করা backward-compatible ধাপে ভেঙে ফেলো। একটা column-এর নাম বদলাতে: **expand** — নতুন column যোগ করো এবং দুটোতেই write করো; **migrate** — বিদ্যমান row-গুলো backfill করো; **transition** — নতুন column পড়ে এমন code deploy করো; **contract** — পুরনোটায় write করা বন্ধ করো এবং পরের একটা release-এ সেটা drop করো। প্রতিটি ধাপ আলাদাভাবেই rollback-এর জন্য নিরাপদ। এতে আরও বেশি কাজ এবং আরও বেশি deploy লাগে, কিন্তু এটা একটা নিশ্চিত-outage অপারেশনকে একটা রুটিন অপারেশনে বদলে দেয়।

### ৪. যে migration একটা বড় table-কে lock করে ফেলে
**আপনি যা দেখেন:** ২০ কোটি row-এর একটা table-এ একটা migration exclusive lock নিয়ে নেয়, query তার পেছনে জমতে থাকে, connection pool শেষ হয়ে যায়, এবং কিছু "ব্যর্থ" না হলেও application কুড়ি মিনিটের জন্য বন্ধ থাকে।

**এটা কেন ঘটে:** কিছু DDL operation পুরো সময়টার জন্যই table lock করে রাখে, এবং সময়টা table-এর আকারের সাথে সাথে বাড়ে।

**এই টপিকটি কীভাবে এটা সমাধান করে:** এটা আপনাকে বাধ্য করে দেখতে যে আপনার নির্দিষ্ট database version-এ প্রতিটি operation আসলে কী করে, এবং নিরাপদ variant ব্যবহার করতে — concurrently index তৈরি করা, defaulted column-এর বদলে nullable column যোগ করা, ছোট ছোট batch-এ pause দিয়ে backfill করা, এবং একটা ছোট lock timeout সেট করা যাতে migration traffic block করার বদলে দ্রুত ব্যর্থ হয়। এটা এমন কয়েকটি জায়গার একটা যেখানে আপনার database-এর সঠিক আচরণ জানা সত্যিই প্রয়োজন।

## যে মূল্য আপনাকে দিতে হয়

- **Backward compatibility একটা চলমান discipline।** প্রতিটি schema পরিবর্তন একটা multi-release sequence হয়ে যায়। কাউকে না কাউকে মনে রাখতে হয় contract ধাপটা চালাতে, আর বাস্তবে cleanup-টাই সবচেয়ে বেশি ভুলে যাওয়া হয়, ফলে বছরের পর বছর dual-write code এবং মৃত column পড়ে থাকে।
- **আরও বেশি deploy, আরও বেশি সমন্বয়।** একটা rename যা এক migration ছিল, সেটা কয়েক দিন ধরে ছড়ানো চারটা release হয়ে যায়।
- **Code-এ সাময়িক জটিলতা।** Dual-writing এবং fallback-সহ read করা কুৎসিত code, যা শুধু একটা transition চলাকালীন সময়েই অস্তিত্বে থাকে, এবং দুটো path আলাদা হয়ে গেলে এটা bug-এর একটা উৎস হয়ে দাঁড়ায়।
- **Infrastructure খরচ।** একটা switch-এর সময় blue-green environment দ্বিগুণ করে দেয়; canary-র জন্য traffic-splitting এবং version-গুলোর মধ্যে metric-এর automated তুলনা দরকার হয়।
- **দীর্ঘ সময় ধরে চলা backfill নিজেই একটা operational কাজ।** এগুলো resumable হতে হবে, database saturate না করার জন্য throttled হতে হবে, এবং monitor করতে হবে — কখনো কখনো দিনের পর দিন ধরে।
- **Rollback সবসময় সম্ভব নয়।** একবার আপনি একটা column drop করে ফেললে বা data ধ্বংসাত্মকভাবে রূপান্তর করে ফেললে, ফিরে যাওয়ার আর কোনো উপায় থাকে না। যে discipline-টা গুরুত্বপূর্ণ তা হলো ধাপগুলোকে এমনভাবে সাজানো যাতে অপরিবর্তনীয় ধাপটা সবার শেষে আসে, এবং তাও কেবল নতুন version স্থিতিশীল হওয়ার পরই।

## কখন এটা দরকার — এবং কখন নয়

| পুরো zero-downtime discipline দরকার যখন | সহজ পদ্ধতিই যথেষ্ট যখন |
|---|---|
| System-টার বিভিন্ন time zone জুড়ে বাস্তব ব্যবহারকারী আছে | সহনশীল, পরিচিত দর্শকসহ internal tool |
| আপনি ঘন ঘন deploy করেন (যা আপনার করা উচিতও) | সত্যিকার অর্থে কম-traffic system, যেখানে window নিয়ে সম্মতি আছে |
| একটা outage-এর আর্থিক বা চুক্তিগত খরচ আছে | এমন একটা pre-launch পণ্য যার কোনো ব্যবহারকারী নেই |
| Table এত বড় যে migration-এ সত্যিই সময় লাগে | Table ছোট এবং migration তাৎক্ষণিক |

**তবে backward-compatible migration একটা default অভ্যাস হিসেবে করার যোগ্য** — খরচ কম, আর যে failure এটা প্রতিরোধ করে তা মারাত্মক এবং আকস্মিক।

## কেন এটা Interview-এ আসে

"আপনি এটা কীভাবে downtime ছাড়া deploy করবেন?" এবং "আপনি কীভাবে এই schema নিরাপদে পরিবর্তন করবেন?" — এই দুটোই ব্যবহারিক প্রশ্ন যা নির্ভরযোগ্যভাবে production অভিজ্ঞতাসম্পন্ন candidate-দের আলাদা করে ফেলে। উচ্চ-signal উত্তরটা হলো health check এবং connection draining-সহ rolling deploy-এর নাম বলা, risk নিয়ন্ত্রণের জন্য canary বা blue-green-এর কথা বলা, এবং — সবচেয়ে গুরুত্বপূর্ণভাবে — ব্যাখ্যা করা যে পুরনো ও নতুন code একসাথে চলে, তাই schema পরিবর্তনকে expand-and-contract-এর মাধ্যমে backward compatible হতে হবে। একটা বড় table-এর migration lock করে site ফেলে দিতে পারে, এবং আপনি সেটা কীভাবে এড়াবেন — এই কথা উল্লেখ করাটা এমন একটা বিস্তারিত জ্ঞান যা কেবল সেটা সরাসরি অভিজ্ঞতা করেই আসে।

## এটি কীভাবে সংযুক্ত

Zero-downtime deploy তৈরি হয় **load balancer**-এর health check এবং draining-এর উপর (topic 7) এবং এগুলো বেশিরভাগ automate হয় **Kubernetes**-এর rolling update এবং readiness probe দিয়ে (topic 44)। Canary analysis সম্পূর্ণভাবে নির্ভর করে **observability**-র উপর (topic 43), যাতে version-গুলো তুলনা করা যায়। Dual-version সমস্যাটা **message format** (topic 35) এবং **microservices communication** (topic 31)-এর API compatibility সংক্রান্ত উদ্বেগেরই একটা রূপ। নিরাপদ migration **replication** lag (topic 13) এবং **isolation level**-এর (topic 37) সাথে interact করে, এবং পুরো এই practice-টাই ঘন ঘন, কম-ঝুঁকির delivery-কে topic 5-এর **availability** target-এর সাথে compatible করে তোলে।

**পরবর্তী:** [Testing Distributed Systems](../46-testing-distributed-systems-load-testing-and-chaos-engineering/why.md) — এসবের কোনোটা আসলে কাজ করে কিনা তা আপনার ব্যবহারকারীদের আগেই জেনে নেওয়া।
