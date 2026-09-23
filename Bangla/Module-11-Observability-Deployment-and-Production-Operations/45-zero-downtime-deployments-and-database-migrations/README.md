# Zero-Downtime Deployments & Database Migrations

**কঠিনতা:** Intermediate/Advanced

## Learning Objectives

- ব্যাখ্যা করা যে কেন downtime ছাড়া একটি service-এর নতুন version রিলিজ করা "পুরনো code কে শুধু replace করে দেওয়া"-র চেয়ে বেশি কঠিন।
- rolling, blue-green এবং canary deployment strategy তুলনা করা এবং তাদের মধ্যকার trade-off বোঝা।
- ব্যাখ্যা করা যে কেন যেকোনো zero-downtime deployment-এর সময় একটি service-এর পুরনো ও নতুন version-কে অল্প সময়ের জন্য একসাথে থাকতে হয়, এবং এর ফলে API compatibility-র জন্য কী দরকার হয়।
- একটি running application-কে না ভেঙে database schema পরিবর্তনের জন্য expand-contract pattern বর্ণনা করা।
- application code এবং database schema — দুটোকেই একসাথে স্পর্শ করে এমন একটি পরিবর্তনের জন্য একটি নিরাপদ rollout plan design করা।

## Script

### Hook / Intro

গত ভিডিওতে আমরা দেখেছি Kubernetes কীভাবে container schedule ও heal করে। আজকের প্রশ্নটা তার চেয়ে সংকীর্ণ, তবে বাস্তব অনেক incident-এ এটাই বেশি বিপজ্জনক: আপনি কীভাবে একটি running service-এর একটি version-কে নতুন version দিয়ে replace করবেন, পুরো সময়টা জুড়ে live production traffic serve করার সময়েও একটিও request না হারিয়ে — এবং একই কাজ আপনার database-এর schema-র সাথেও কীভাবে করবেন, যাকে একটি container-এর মতো "restart" করে ফেলা যায় না। এটা ভুল করলে আপনি পাবেন সেই ক্লাসিক রাত ২টার incident: staging-এ একদম ঠিকঠাক দেখানো একটা deploy production-এ checkout আট মিনিটের জন্য বন্ধ করে দেয়।

### কেন "শুধু Replace করে দাও" কাজ করে না

আপনি যদি পুরনো version-এর সব instance একসাথে বন্ধ করে নতুন version চালু করেন, তাহলে অনিবার্যভাবে একটা ফাঁক তৈরি হবে — যত ছোটই হোক না কেন — যেখানে request serve করার জন্য কোনো instance-ই available থাকবে না, এবং ঠিক সেই মুহূর্তে চলমান প্রতিটি request ব্যর্থ হবে। বাস্তব scale-এ, "যত ছোটই হোক না কেন" পরিমাণটাও এত request ব্যর্থ করার জন্য যথেষ্ট যে সেটা একটা দৃশ্যমান, বাস্তব incident হয়ে দাঁড়ায়। Zero-downtime deployment strategy-গুলো ঠিক এই ফাঁকটা দূর করার জন্যই তৈরি — এগুলো নিশ্চিত করে যে *healthy, ready* instance-এর মোট সংখ্যা কখনোই শূন্যে নেমে না যায় — এবং এটা গত ভিডিওর readiness probe ও Service abstraction-এর উপর ভিত্তি করেই তৈরি।

### Rolling Deployments

একটি **rolling deployment** — যা Kubernetes Deployments-এর default — পুরনো instance-গুলোকে একবারে অল্প কয়েকটা করে নতুন instance দিয়ে replace করে: প্রথমে একটা নতুন-version pod চালু করো, তার readiness probe pass করার জন্য অপেক্ষা করো, তারপর একটা পুরনো-version pod বন্ধ করো, এবং সব instance নতুন version-এ না আসা পর্যন্ত এটা পুনরাবৃত্তি করো। এই পুরো প্রক্রিয়ার প্রতিটি মুহূর্তে, ready instance-এর মোট pool মোটামুটি স্থির থাকে, ফলে Service পুরো সময় জুড়েই সফলভাবে traffic route করতে পারে। এখানে আসল সূক্ষ্মতাটা — যেটা অপ্রস্তুত team-গুলোকে ধরে ফেলে — তা হলো: rollout-এর পুরো সময় জুড়েই *পুরনো এবং নতুন — দুই version একসাথে live traffic serve করে*। যদি নতুন version কোনো API response-এর গঠন, database column-এর অর্থ, বা queue-তে থাকা কোনো message format এমনভাবে পরিবর্তন করে যা পুরনো version বুঝতে পারে না (বা উল্টোটা), তাহলে rollout window-এর সময় আপনি সত্যিকারের, live error পাবেন — এই কারণেই backward-compatible, additive পরিবর্তন (নতুন optional field, রিনেম বা মুছে ফেলা field নয়) যেকোনো rolling-deploy করা service-এর জন্য একটা standard discipline, যা Module 8-এর Protocol Buffers ভিডিওর schema-evolution নীতিরই প্রতিধ্বনি।

### Blue-Green এবং Canary Deployments

একটি **blue-green deployment** ভিন্ন পদ্ধতি নেয়: বর্তমান environment-এর ("blue") পাশাপাশি একটি সম্পূর্ণ, পুরোপুরি scale করা দ্বিতীয় environment ("green") চালাও, blue যখন সব live traffic serve করছে তখন নতুন version পুরোপুরি green-এ deploy করো, এবং green health check pass করলে router/load balancer-কে সব traffic এক atomic cut-over-এ green-এ পাঠাতে সুইচ করো। যদি কিছু ভুল হয়, rollback মানে শুধু router-কে আবার blue-তে ফিরিয়ে দেওয়া, যা তখনও সম্পূর্ণভাবে চলমান এবং অস্পৃষ্ট — rolling deployment-এর তুলনায় এটা অনেক দ্রুত এবং পরিষ্কার একটা rollback, কারণ rolling deployment-এ ধীরে ধীরে rollback করতে হয় অথবা পুরনো version পড pড করে পুনরায় deploy করতে হয়। এর খরচ হলো — অন্তত কিছুক্ষণের জন্য — দুটো সম্পূর্ণ production-scale environment একসাথে চালানো, যা সুইচ করার সময়টুকুর জন্য infrastructure খরচ প্রায় দ্বিগুণ করে দেয়।

একটি **canary deployment** আরও সতর্ক পথে এগোয়: প্রথমে নতুন version-কে traffic-এর একটা ছোট অংশে ছাড়ো — ধরা যাক ৫% — এবং error rate, latency, ও অন্যান্য metric (Module 11-এর observability tool-গুলো মনে করুন) খুব ঘনিষ্ঠভাবে পর্যবেক্ষণ করো, তারপর ধীরে ধীরে সেই percentage বাড়িয়ে ১০০%-এ নিয়ে যাও। এতে একটা খারাপ deploy-এর blast radius সবার বদলে ব্যবহারকারীদের একটা ছোট অংশে সীমাবদ্ধ থাকে, তবে এর বিনিময়ে rollout ধীর ও ক্রমবর্ধমান হয় এবং প্রতিটি ধাপে এগোনোর সিদ্ধান্ত নেওয়ার জন্য প্রকৃত পর্যবেক্ষণ ও operational overhead লাগে — আজকাল যা প্রায়ই progressive-delivery tooling দিয়ে automate করা হয়, যা নিজে থেকেই metric দেখে canary percentage এগিয়ে নেয় (বা প্রয়োজনে automatic rollback করে)।

### Database Migrations: Expand-Contract Pattern

Application code তুলনামূলকভাবে পরিষ্কারভাবে blue-green বা canary করা যায় কারণ আপনি দুটো version পাশাপাশি চালাতে পারেন। একটা database schema আরও জটিল — সাধারণত একটাই database থাকে, যা যেকোনো rollout-এর সময় পুরনো ও নতুন উভয় application version-ই শেয়ার করে — তাই একটা schema পরিবর্তন *দুই* version-এরই একসাথে read এবং write করার জন্য নিরাপদ হতে হবে, অন্তত deployment-এর পুরো সময়টুকুর জন্য। এর জন্য standard discipline হলো **expand-contract pattern**, যা একটা atomic পরিবর্তনের বদলে আলাদা, ধারাবাহিক deploy-তে করা হয়:

**Expand**: পুরনো কিছু না সরিয়ে বা পুনর্ব্যবহার না করে নতুন schema element (নতুন column, table, বা index) যোগ করো — পুরনো application code সেই নতুন column, যেটা সে চেনে না, সেটাকে সহজেই উপেক্ষা করে, ঠিক Protocol Buffers-এর additive schema-evolution নীতির মতো। শুধু এই migration-টাই deploy করো, এখনও application code-এ কোনো পরিবর্তন ছাড়াই। **Migrate**: application-এর একটা নতুন version deploy করো যা পুরনো এবং নতুন — *উভয়* column/table-এই write করে (এবং যেটা এখন authoritative সেখান থেকে read করে), এবং প্রয়োজনমতো পুরনো data নতুন structure-এ backfill করে। **Contract**: একবার নিশ্চিত হলে যে প্রতিটি instance নতুন schema ব্যবহারকারী version চালাচ্ছে, এবং পুরনো data সম্পূর্ণভাবে backfill হয়ে গেছে, একটা চূড়ান্ত পরিবর্তন deploy করো যা পুরনো column/table-এ write করা বন্ধ করে দেয়, এবং তারপরই — পরে একটা cleanup ধাপে — সেটা আসলে drop করো। এক ধাক্কায় "এই column-এর নাম বদলে দাও এবং নতুন নামের প্রত্যাশা রাখা নতুন code deploy করো" করার চেষ্টা করাই ঠিক সেই ভুল যা একটা rolling deployment ভেঙে দেয় — rollout window-এর সময় তখনও চলমান পুরনো-version pod-গুলো সাথে সাথেই এমন একটা column-এর বিরুদ্ধে ব্যর্থ হতে শুরু করবে যেটা পুরনো নামে আর অস্তিত্বেই নেই।

### বাস্তব উদাহরণ

কল্পনা করুন আপনি `user.name` column-এর নাম বদলে `user.full_name` করছেন, এমন একটা service-এ যেখানে rolling deployment একটা standard practice। এটাকে একটা মাত্র পরিবর্তন হিসেবে করার চেষ্টা করলে — column migrate করা এবং শুধু `full_name` পড়ে এমন নতুন code deploy করা — সাথে সাথেই ভেঙে পড়বে, কারণ rollout-এর মাঝখানে তখনও চলমান পুরনো-version pod-গুলো তখনও `name` query করতে থাকবে, যা আর অস্তিত্বেই নেই। এর বদলে expand-contract ব্যবহার করলে: প্রথম deploy অস্পৃষ্ট `name` column-এর পাশাপাশি একটা নতুন `full_name` column যোগ করে (expand); দ্বিতীয় deploy এমন application code পাঠায় যা প্রতিটি update-এ উভয় column-এই write করে এবং বিদ্যমান row-গুলোর `full_name` কে `name` থেকে backfill করে; সম্পূর্ণভাবে rollout ও backfill হয়ে গেলে, তৃতীয় deploy সব read `full_name`-এ সুইচ করে এবং `name`-এ write করা বন্ধ করে দেয়; এটা কিছুদিন নিরাপদভাবে চলার পরই একটা চূড়ান্ত cleanup migration আসলে এখন-অব্যবহৃত `name` column drop করে। একটা deploy-এর বদলে চারটা আলাদা, প্রতিটি নিজে থেকেই নিরাপদ ধাপ — যেখানে একটা মাত্র deploy একটা বাস্তব incident ঘটিয়ে ফেলতে পারত।

### Recap

Zero-downtime deployment strategy-গুলো এই উদ্দেশ্যেই তৈরি যাতে healthy, ready instance-এর pool কখনো শূন্যে না পৌঁছায়, কিন্তু এগুলোর সবগুলোই (rolling, blue-green, canary) rollout-এর অন্তত কিছু অংশের জন্য পুরনো ও নতুন application version-কে একসাথে থাকতে এবং সঠিকভাবে traffic serve করতে বাধ্য করে — যার মানে সেই সময়টুকুর জন্য পরিবর্তনগুলো backward-compatible হতে হবে। Rolling deployment খুব কম বাড়তি infrastructure খরচে ধীরে ধীরে instance replace করে; blue-green পুরো environment atomically সুইচ করে, infrastructure খরচের বিনিময়ে একটা দ্রুত, পরিষ্কার rollback দেয়; canary metric দেখতে দেখতে ধীরে ধীরে traffic বাড়িয়ে একটা খারাপ deploy-এর blast radius সীমিত রাখে। Database schema পরিবর্তনেও একই "পুরনো ও নতুনকে একসাথে থাকতে হবে" discipline দরকার, যা expand-contract pattern হিসেবে formal করা হয়েছে: আগে নতুন structure যোগ করো, তারপর application code এবং data migrate করো, এবং কারো উপর নির্ভরতা না থাকলে তবেই পুরনো structure সরাও।

### এরপর কী

আমরা এখন নিরাপদে পরিবর্তন deploy করা নিয়ে আলোচনা করেছি। পরের ভিডিওতে একটা সম্পর্কিত কিন্তু ভিন্ন প্রশ্ন তোলা হবে: আপনি আসলে কীভাবে — ইচ্ছাকৃতভাবে এবং সক্রিয়ভাবে — *প্রমাণ* করবেন যে আপনার system বাস্তব load এবং বাস্তব failure সহ্য করতে পারে — একটা প্রকৃত incident-এর সময় প্রথমবার সেটা জানার বদলে?

## Key Takeaways

- Zero-downtime deployment strategy-গুলো healthy, ready instance-এর pool কখনো শূন্যে নামতে না দিয়ে কাজ করে, যা readiness probe এবং load-balanced service discovery-র উপর ভিত্তি করে তৈরি।
- Rolling deployment ধীরে ধীরে এবং সস্তায় instance replace করে, কিন্তু এর মানে rollout-এর সময় পুরনো ও নতুন version একসাথে traffic serve করে — যার জন্য backward-compatible পরিবর্তন প্রয়োজন।
- Blue-green deployment একটা সম্পূর্ণ দ্বিতীয় environment চালায় এবং atomically cut over করে, যা সাময়িকভাবে দ্বিগুণ infrastructure-এর বিনিময়ে একটা দ্রুত, পরিষ্কার rollback দেয়।
- Canary deployment metric দেখতে দেখতে নতুন-version traffic ধীরে ধীরে বাড়ায়, যা ধীরগতির rollout-এর বিনিময়ে একটা খারাপ deploy-এর blast radius সীমিত করে।
- Expand-contract pattern এই যেকোনো strategy-র অধীনে database schema পরিবর্তনকে নিরাপদ করে তোলে: আগে নতুন structure যোগ করো (expand), সেটা ব্যবহার করার জন্য code/data migrate করো, তারপর কারো উপর নির্ভরতা না থাকলে তবেই পুরনো structure সরাও (contract) — কখনোই এক atomic ধাপে রিনেম/মুছে ফেলা নয়।
