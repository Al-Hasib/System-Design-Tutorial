# অনুশীলন ও Interview প্রশ্ন (Practice & Interview Questions)

**১. কেন একটি single region-এর মধ্যে redundancy একটি critical সিস্টেমের জন্য যথেষ্ট সুরক্ষা নয়?**
একটি region-এর মধ্যে redundancy (একাধিক server, availability zone, replicated database) সেই region-এর মধ্যে মেশিন, rack, এমনকি একক data-center failure-এর বিরুদ্ধে সুরক্ষা দেয়। পুরো region অগম্য হয়ে গেলে এটি কোনো কাজে আসে না — power outage, networking failure, বা ভাগ করা regional infrastructure-কে প্রভাবিত করা কোনো দুর্যোগের কারণে — যা একটি বিরল কিন্তু বাস্তব ঝুঁকি, যা শুধুমাত্র multi-region architecture মোকাবিলা করে।

**২. active-passive multi-region architecture এবং এর দুটি প্রধান খরচ বর্ণনা করুন।**
একটি region (active) সব live traffic পরিবেশন করে, আর দ্বিতীয় একটি region (passive/standby) data-এর একটি replicated copy রাখে কিন্তু স্বাভাবিক অবস্থায় production traffic পরিবেশন করে না; active region fail করলে, traffic passive region-এ fail over করে। দুটি খরচ হলো idle capacity (standby region-এর জন্য অর্থ দেওয়া হয় কিন্তু বেশিরভাগ সময় ব্যবহৃত হয় না) এবং failover delay (traffic redirect করা, warm-up করা, data-এর সাম্প্রতিকতা যাচাই করা — সবগুলোতেই প্রকৃত সময় লাগে)।

**৩. active-active architecture কীভাবে active-passive-এর দুটি প্রধান খরচই দূর করে, এবং এর বিনিময়ে কোন খরচ চালু করে?**
প্রতিটি region একই সাথে live traffic পরিবেশন করে, তাই কোনো idle standby capacity নেই এবং কোনো failover delay নেই (কোনো একক active region "take over" করার জন্য প্রয়োজন নেই)। এটি যে খরচ চালু করে তা হলো cross-region data consistency: যেহেতু একাধিক region-এ একই সাথে write গ্রহণ করা যায়, সিস্টেমকে প্রতিটি write-এ সরাসরি CAP/PACELC trade-off-এর মুখোমুখি হতে হয়।

**৪. RTO এবং RPO সংজ্ঞায়িত করুন, এবং ব্যাখ্যা করুন কেন এগুলোকে নিছক টেকনিক্যাল নয় বরং ব্যবসায়িক সিদ্ধান্ত হিসেবে বর্ণনা করা হয়।**
RTO (Recovery Time Objective) হলো একটি disaster এবং সিস্টেম আবার অনলাইনে ফিরে আসার মধ্যে সর্বোচ্চ গ্রহণযোগ্য সময়। RPO (Recovery Point Objective) হলো সর্বোচ্চ গ্রহণযোগ্য data loss-এর পরিমাণ, সময়ে পরিমাপ করা হয়। এগুলো ব্যবসায়িক সিদ্ধান্ত কারণ "সঠিক" মান নির্ভর করে শক্তিশালী guarantee-র খরচ (latency, infrastructure, engineering জটিলতা) সেই নির্দিষ্ট সিস্টেমের downtime বা data loss-এর প্রকৃত ব্যবসায়িক খরচের বিপরীতে ওজন করার উপর — এখানে কোনো সার্বজনীনভাবে সঠিক সংখ্যা নেই।

**৫. শূন্য RPO (কোনো গ্রহণযোগ্য data loss নেই) কীভাবে একটি সিস্টেম যে replication স্ট্র্যাটেজি ব্যবহার করতে পারে তা সীমাবদ্ধ করে?**
শূন্য RPO-র জন্য synchronous cross-region replication প্রয়োজন — সম্পূর্ণ হিসেবে বিবেচিত হওয়ার আগে প্রতিটি write অন্য একটি region-এ replicate হয়েছে বলে নিশ্চিত করতে হবে — যা region-গুলোর মধ্যে network দূরত্ব/অবস্থার সমানুপাতিক প্রকৃত latency প্রতিটি write-এ যোগ করে, এই guarantee-র বিনিময়ে যে কোনো নিশ্চিত write কখনোই হারানো যাবে না, এমনকি একটি region সাথে সাথেই fail করলেও।

**৬. শুধুমাত্র কয়েক সেকেন্ডের একটি RTO পূরণ করতে সাধারণত কেন active-active architecture প্রয়োজন হবে?**
যেকোনো architecture যাতে failover জড়িত — একটি region থেকে অন্যটিতে traffic redirect করা — কিছুটা প্রকৃত সময় নেয় (DNS propagation, application warm-up, verification), যা বাস্তবসম্মতভাবে শুধু সেকেন্ডে কমানো যায় না। Active-active এই সমস্যা সম্পূর্ণভাবে এড়িয়ে যায় কারণ প্রথম থেকেই কোনো একক active region নেই যা থেকে fail over করতে হবে; অন্য region-গুলো ইতিমধ্যেই traffic পরিবেশন করছে।

**৭. একই কোম্পানির মধ্যে দুটি ভিন্ন সিস্টেমের যুক্তিসঙ্গতভাবে খুব ভিন্ন RTO/RPO লক্ষ্য কেন থাকতে পারে?**
কারণ downtime এবং data loss-এর ব্যবসায়িক খরচ সিস্টেম ভেদে ভিন্ন হয় — একটি internal analytics dashboard-এর কয়েক ঘণ্টা ডাউন থাকা বা একদিনের data হারানোর খরচ কম, আর একটি customer-facing payment system-এর এক মিনিট downtime বা যেকোনো হারানো transaction-এর খরচ অনেক বেশি। RTO/RPO-কে প্রতিটি সিস্টেমের প্রকৃত ব্যবসায়িক প্রভাব প্রতিফলিত করা উচিত, একটি একক কোম্পানি-ব্যাপী default নয়।

**৮. পরিস্থিতি: একটি কোম্পানির তাদের core transaction database-এর জন্য শূন্য data loss এবং প্রায়-তাৎক্ষণিক recovery প্রয়োজন, কিন্তু তারা এর জন্য প্রয়োজনীয় অতিরিক্ত write latency কমাতে চায়। এটি কি অর্জনযোগ্য, এবং আপনি তাদের কী বলবেন?**
এটি একটি সরাসরি trade-off, optimize করে দূর করার মতো কিছু নয় — শূন্য data loss (RPO শূন্য)-এর জন্য synchronous cross-region replication প্রয়োজন, যা স্বাভাবিকভাবেই inter-region network অবস্থার সমানুপাতিক latency প্রতিটি write-এ যোগ করে। আপনি সেই latency কমাতে পারেন শুধুমাত্র ভৌগোলিকভাবে কাছাকাছি region বেছে নিয়ে বা asynchronous replication-এর মাধ্যমে একটি শূন্য নয় (কিন্তু ছোট) RPO মেনে নিয়ে — আপনি শূন্য অতিরিক্ত latency সহ শূন্য RPO পেতে পারবেন না; এটাই আসল trade-off যা করা হচ্ছে।

**৯. পরিস্থিতি: একটি বিরল regional outage-এর পর মুষ্টিমেয় কয়েকজন analyst-এর ব্যবহৃত একটি internal reporting tool কয়েক ঘণ্টার জন্য ডাউন হয়ে যায়। এখানে কি একটি multi-region active-active architecture স্পষ্টভাবে ন্যায্য?**
অগত্যা নয় — সীমিত ব্যবহারকারী সহ একটি internal tool-এর জন্য কয়েক ঘণ্টার downtime-এর কম ব্যবসায়িক খরচ বিবেচনায়, backup থেকে একটি documented manual recovery process (একটি উচ্চ RTO/RPO সহনশীলতা) সম্পূর্ণভাবে গ্রহণযোগ্য হতে পারে, এবং active-active multi-region architecture-এর অতিরিক্ত খরচ ও জটিলতা সম্ভবত এই নির্দিষ্ট সিস্টেমের জন্য ন্যায্য নয়।

**১০. সত্য না মিথ্যা: Active-active architecture single-region সিস্টেমকে যে CAP theorem trade-off করতে হয় তা এড়িয়ে যায়।**
মিথ্যা। Active-active architecture CAP/PACELC trade-off এড়িয়ে যায় না — এটি সেগুলোকে অনিবার্য এবং স্পষ্ট করে তোলে, কারণ এখন একাধিক region-এ একই সাথে write গ্রহণ করা যায়, যা strong cross-region consistency (এর latency/availability খরচ সহ) এবং eventual consistency (এর conflict-resolution প্রয়োজনীয়তা সহ)-এর মধ্যে একটি সরাসরি পছন্দে বাধ্য করে, যা single-region সিস্টেমকে region স্তরে একেবারেই মোকাবিলা করতে হয় না।
