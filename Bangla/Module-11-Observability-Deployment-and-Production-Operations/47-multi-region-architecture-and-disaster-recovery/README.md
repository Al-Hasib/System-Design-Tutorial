# Multi-Region Architecture & Disaster Recovery

**কঠিনতার স্তর:** Advanced

## শেখার উদ্দেশ্য (Learning Objectives)

- একটি single region-এর মধ্যে redundancy কেন প্রতিটি বাস্তব failure scenario-র বিরুদ্ধে যথেষ্ট সুরক্ষা নয়, তা ব্যাখ্যা করা।
- active-passive এবং active-active multi-region architecture এবং তাদের trade-off তুলনা করা।
- RTO এবং RPO সংজ্ঞায়িত করা এবং ব্যাখ্যা করা কেন এগুলো নিছক টেকনিক্যাল নয়, বরং ব্যবসায়িক সিদ্ধান্ত।
- multi-region write availability যে নির্দিষ্ট চ্যালেঞ্জগুলো তৈরি করে তা ব্যাখ্যা করা, এবং তা CAP/PACELC-এর সাথে সংযুক্ত করা।
- একটি সিস্টেমের প্রকৃত availability এবং consistency প্রয়োজনীয়তা অনুযায়ী উপযুক্ত disaster recovery স্ট্র্যাটেজি ডিজাইন করা।

## স্ক্রিপ্ট (Script)

### শুরুর কথা (Hook / Intro)

Module 1-এ আমরা redundancy নিয়ে আলোচনা করেছিলাম: একাধিক server, একাধিক database replica, যাতে একটি মেশিন fail করলেও পুরো সিস্টেম ডাউন না হয়। এটি প্রকৃত সুরক্ষা — কিন্তু একক মেশিনের fail হওয়ার বিরুদ্ধে। যে facility-তে সেই মেশিনটি থাকে, তা যদি পুরোপুরি power হারায়, আগুন লাগে, বা data center বা cloud region পর্যায়ে networking failure-এর কারণে অগম্য হয়ে পড়ে, তখন এটি কোনো কাজে আসে না — এটি বিরল ঘটনা, কিন্তু কাল্পনিক নয়; প্রতিটি major cloud provider-এরই region-level outage হয়েছে। আজ আমরা তার পরের স্তর নিয়ে আলোচনা করব: multi-region architecture, এবং disaster recovery planning, যা নির্ধারণ করে ঠিক কতটা data loss এবং downtime আপনার ব্যবসা প্রকৃতপক্ষে মেনে নিতে রাজি, যখন সেই অসম্ভাব্য ঘটনাটি ঘটে যায়।

### কেন একটি Region-এর মধ্যে Redundancy যথেষ্ট নয়

cloud infrastructure-এর পরিভাষায় একটি "region" হলো ভৌগোলিকভাবে স্বতন্ত্র একটি অবস্থান, যা অন্য region থেকে শারীরিকভাবে বিচ্ছিন্ন, ঠিক এই কারণেই যাতে একটিতে ঘটা failure (power, networking, প্রাকৃতিক দুর্যোগ, ভাগ করা regional infrastructure-কে প্রভাবিত করা মানবিক ভুল) অন্যটিতে ছড়িয়ে না পড়ে। একটি region-এর মধ্যে redundancy — একাধিক availability zone, একাধিক server, replicated database — মেশিন বা rack স্তরের failure-এর বিরুদ্ধে সুরক্ষা দেয়, এমনকি একটি region-এর মধ্যে একটি data center-এর স্তর পর্যন্তও। কিন্তু এটি পুরো region-কে একসাথে প্রভাবিত করে এমন কিছুর বিরুদ্ধে সুরক্ষা দেয় না। multi-region protection তার খরচের যোগ্য কিনা তা সম্পূর্ণভাবে ব্যবসার ওপর নির্ভর করে: একটি personal blog কয়েক ঘণ্টা regional downtime সহ্য করতে পারে; একটি payment processor, একটি healthcare system, বা global e-commerce platform সাধারণত পারে না।

### Active-Passive: একটি Standby Region

সরল multi-region প্যাটার্নটি হলো **active-passive** (একে active-standby-ও বলা হয়): একটি region ("active" বা "primary") সব live traffic পরিবেশন করে, আর দ্বিতীয় একটি region ("passive" বা "standby") replicated data নিয়ে প্রস্তুত থাকে, কিন্তু স্বাভাবিক অবস্থায় production traffic পরিবেশন করে না। active region fail করলে, আপনি **fail over** করেন — traffic redirect করেন passive region-এ, যা এখন active হয়ে যায়। ধারণাগতভাবে এটি সরল এবং multi-region write-এর সবচেয়ে কঠিন সমস্যাগুলো এড়িয়ে যায় (এই নিয়ে একটু পরে আরও বলা হবে), কিন্তু এর দুটি বাস্তব খরচ আছে: standby region-এর capacity বেশিরভাগ সময় অলস থাকে, অর্থ দেওয়া হয় কিন্তু বেশিরভাগ সময় ব্যবহৃত হয় না, এবং failover নিজেই সময় নেয় — DNS propagation, application warm-up, data current কিনা তা যাচাই করা — এই কারণেই disaster recovery planning সেই ফাঁকের জন্য নির্দিষ্ট target সংখ্যা নির্ধারণ করে, "যতক্ষণ লাগে ততক্ষণ" হিসেবে রেখে দেওয়ার বদলে।

### Active-Active: প্রতিটি Region Traffic পরিবেশন করে

**Active-active** আরও এগিয়ে যায়: একাধিক region একই সাথে live production traffic পরিবেশন করে, সাধারণত ভৌগোলিকভাবে নিকটতম region-এ route করা হয় কম latency-র জন্য (Module 4-এর CDN/anycast routing ধারণার প্রতিধ্বনি)। একটি region fail করলে, বাকিগুলো কেবল তার traffic শুষে নেয় — কোনো failover delay নেই, কারণ প্রথম থেকেই কোনো একক active region ছিল না যা থেকে fail over করতে হবে, এবং idle-standby-capacity-র খরচও মূলত অদৃশ্য হয়ে যায় কারণ প্রতিটি region-এর capacity সবসময় প্রকৃত কাজ করছে। খরচটি সম্পূর্ণ ভিন্ন জায়গায় দেখা দেয়: **data consistency**। যদি দুটি ভিন্ন region-এর ব্যবহারকারীরা একই সময়ে একই logical data-য় write করতে পারেন, তাহলে আপনি আবার Module 3-এর CAP theorem এবং PACELC trade-off-এ ফিরে যাচ্ছেন — আপনি কি region জুড়ে strong consistency জোরদার করবেন (প্রতিটি write-এ cross-region coordination-এর latency খরচ মেনে নিয়ে, এবং region-এর মধ্যে network partition-এর সময় কমে যাওয়া availability), নাকি eventual consistency মেনে নেবেন (দ্রুত, সবসময় available local write, কিন্তু একই conflict-detection tools প্রয়োজন — এই module-এর আগের অংশ থেকে vector clocks মনে করুন — যা region জুড়ে সত্যিকারের সমসাময়িক, পরস্পরবিরোধী write সমাধান করতে পারে)। active-active-এর এমন কোনো সংস্করণ নেই যা এই trade-off এড়িয়ে যায়; একাধিক জায়গায় write গ্রহণ করার এটাই প্রত্যক্ষ, অনিবার্য খরচ।

### RTO এবং RPO: "কতটা খারাপ" তা নির্ধারণ করা সংখ্যাগুলো

Disaster recovery planning দুটি নির্দিষ্ট metric-কে ঘিরে formalize করা হয়, এবং এগুলো ঠিকভাবে নির্ধারণ করা মূলত একটি ব্যবসায়িক সিদ্ধান্ত, নিছক টেকনিক্যাল নয়। **RTO (Recovery Time Objective)** হলো একটি disaster ঘটা এবং সিস্টেম আবার চালু হয়ে traffic পরিবেশন শুরু করার মধ্যে সর্বোচ্চ গ্রহণযোগ্য সময় — "আমাদের অবশ্যই 15 মিনিটের মধ্যে আবার অনলাইনে ফিরতে হবে।" **RPO (Recovery Point Objective)** হলো সর্বোচ্চ গ্রহণযোগ্য data loss-এর পরিমাণ, সময়ে পরিমাপ করা হয় — "আমরা সর্বোচ্চ সাম্প্রতিক 5 মিনিটের write হারাতে পারি।" এই দুটি সংখ্যা সরাসরি architecture সিদ্ধান্ত চালিত করে: শূন্য RPO (কোনো data loss গ্রহণযোগ্য নয়) প্রতিটি write-এ synchronous cross-region replication দাবি করে, সেই guarantee-র বিনিময়ে প্রতিটি একক write-এ প্রকৃত latency দিয়ে; কয়েক মিনিটের RPO asynchronous replication সহ্য করতে পারে, যা প্রতিদিনের জন্য দ্রুততর এবং সাশ্রয়ী, এই সম্ভাবনা মেনে নিয়ে যে disaster ঘটার ঠিক মুহূর্তে সাম্প্রতিকতম কিছু write হয়তো এখনো replicate হয়নি। একইভাবে, সেকেন্ডের RTO কার্যত active-active দাবি করে (কোনো failover delay নেই একেবারেই); ঘণ্টার RTO active-passive দিয়ে পূরণ করা যায়, বা সবচেয়ে কম দাবিদার ক্ষেত্রে, backup থেকে একটি documented manual recovery process দিয়েও। কোনো সংখ্যারই সার্বজনীনভাবে "সঠিক" উত্তর নেই — এগুলো নির্ধারিত হয় শক্তিশালী guarantee-র খরচকে সেই নির্দিষ্ট সিস্টেমের জন্য downtime বা data loss-এর প্রকৃত ব্যবসায়িক খরচের বিপরীতে ওজন করে।

### বাস্তব-জগতের উদাহরণ

একটি কোম্পানির কথা ভাবুন যে একই cloud provider-এ একটি internal analytics dashboard (কম গুরুত্বপূর্ণ) এবং তার customer-facing payment processing system (উচ্চ গুরুত্বপূর্ণ) চালাচ্ছে। analytics dashboard-এর যুক্তিসঙ্গতভাবে RTO "কয়েক ঘণ্টা" এবং RPO "এক দিন পর্যন্ত" হতে পারে — একটি documented, বেশিরভাগ-manual backup থেকে recovery প্রক্রিয়া এখানে সত্যিই ঠিক আছে, কারণ একটি internal tool-এর জন্য ধীর, lossy recovery-র ব্যবসায়িক খরচ কম। payment system-এর RTO "এক মিনিটের কম" এবং RPO "শূন্য data loss" হতে পারে — বিশেষভাবে transaction data-র জন্য synchronous cross-region replication সহ active-active architecture দাবি করে, যদিও এটি প্রতিটি payment write-এ প্রকৃত latency যোগ করে, কারণ একটি নিশ্চিত transaction হারানো, বা business hours-এ এক মিনিটের বেশি সময় ডাউন থাকার ব্যবসায়িক খরচ অগ্রহণযোগ্যভাবে বেশি। একই কোম্পানি, একই cloud provider, দুটি সম্পূর্ণ ভিন্ন, ইচ্ছাকৃতভাবে বেছে নেওয়া architecture — দুটি ভিন্ন সেট RTO/RPO সংখ্যা দ্বারা চালিত, যা ব্যবসার কেউ, শুধু engineering নয়, স্পষ্টভাবে অনুমোদন করেছে।

### সারসংক্ষেপ (Recap)

একটি single region-এর মধ্যে redundancy মেশিন এবং data-center-স্তরের failure-এর বিরুদ্ধে সুরক্ষা দেয়, কিন্তু পুরো একটি region ডাউন হয়ে যাওয়ার বিরুদ্ধে নয় — একটি বিরল কিন্তু বাস্তব ঝুঁকি, যা মোকাবিলা করার জন্যই multi-region architecture-এর অস্তিত্ব। Active-passive একটি standby region প্রস্তুত রাখে এবং প্রয়োজনে fail over করে, সরল কিন্তু idle capacity খরচ এবং failover delay সহ। Active-active প্রতিটি region থেকে একই সাথে traffic পরিবেশন করে, failover delay এবং idle capacity দূর করে, কিন্তু cross-region write consistency trade-off-এর মুখোমুখি সরাসরি হতে হয় (CAP/PACELC trade-off, একাধিক জায়গায় write ঘটলে যা অনিবার্য)। RTO এবং RPO হলো নির্দিষ্ট, ব্যবসা-চালিত সংখ্যা — সর্বোচ্চ গ্রহণযোগ্য downtime এবং সর্বোচ্চ গ্রহণযোগ্য data loss — যা নির্ধারণ করে একটি নির্দিষ্ট সিস্টেমের প্রকৃতপক্ষে কোন architecture (এবং কোন replication স্ট্র্যাটেজি) প্রয়োজন, এবং এগুলো ইচ্ছাকৃতভাবে নির্ধারণ করা উচিত, অন্তর্নিহিত রেখে দেওয়া নয়।

### পরবর্তী কী (What's Next)

observability, deployment, এবং production operations নিয়ে এই module-এর মাধ্যমে শেষ হয় — এবং এর সাথে এই কোর্সের deep-dive content-ও। এখান থেকে, Module 12-এর সম্পূর্ণ, end-to-end case study-তে সবকিছু একত্রিত হবে, একটি ক্লাসিক warm-up দিয়ে শুরু হবে: একটি খালি whiteboard থেকে URL shortener ডিজাইন করা।

## মূল বিষয়গুলো (Key Takeaways)

- একটি region-এর মধ্যে redundancy মেশিন/data-center failure-এর বিরুদ্ধে সুরক্ষা দেয়, কিন্তু region-স্তরের failure-এর বিরুদ্ধে নয় — multi-region architecture বিশেষভাবে সেই বিরল, উচ্চ-প্রভাবশালী ঝুঁকির জন্য বিদ্যমান।
- Active-passive একটি standby region প্রস্তুত রাখে, active region fail করলে fail over করে — সরল, কিন্তু idle standby capacity খরচ এবং প্রকৃত failover delay সহ।
- Active-active প্রতিটি region থেকে একই সাথে traffic পরিবেশন করে, failover delay দূর করে, কিন্তু cross-region write-এ একটি স্পষ্ট CAP/PACELC trade-off বাধ্য করে, কারণ data একই সময়ে একাধিক জায়গায় write করা যেতে পারে।
- RTO (সর্বোচ্চ গ্রহণযোগ্য downtime) এবং RPO (সর্বোচ্চ গ্রহণযোগ্য data loss) হলো ব্যবসা-চালিত সংখ্যা যা সরাসরি নির্ধারণ করে সঠিক architecture এবং replication স্ট্র্যাটেজি — সার্বজনীন constant নয়।
- একই কোম্পানির মধ্যেও সিস্টেম ভেদে সঠিক multi-region/DR স্ট্র্যাটেজি ভিন্ন হয় — প্রতিটি সিস্টেমের downtime এবং data loss-এর প্রকৃত ব্যবসায়িক খরচ দ্বারা চালিত, একটি one-size-fits-all default দ্বারা নয়।
