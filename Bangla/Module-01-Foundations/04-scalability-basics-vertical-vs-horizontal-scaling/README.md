# Scalability Basics: Vertical vs Horizontal Scaling

**Difficulty:** Beginner

## Learning Objectives

- Scalability সংজ্ঞায়িত করুন এবং ব্যাখ্যা করুন কেন এটি system design-এর একটি কেন্দ্রীয় বিষয়।
- Vertical scaling (scaling up) থেকে horizontal scaling (scaling out)-কে আলাদা করুন।
- প্রতিটি scaling strategy-র বাস্তব সীমাবদ্ধতা ও trade-off বুঝুন।
- বুঝুন কেন horizontal scaling load balancing এবং statelessness-এর প্রয়োজনীয়তা তৈরি করে (Module 2-এর পূর্বপ্রদর্শন)।
- একটি সিস্টেমে এমন সংকেত চিহ্নিত করুন যা নির্দেশ করে কখন scale করার সময় এসেছে, এবং কোন দিকে scale করা উচিত।

## Script

### Hook / Intro

আগের video-তে আমরা আপনার browser থেকে একটি server পর্যন্ত এবং আবার ফিরে আসা একটি একক request-এর গতিপথ অনুসরণ করেছিলাম। কিন্তু যখন এটি এক ব্যবহারকারীর এক request নয় — বরং দশ লক্ষ ব্যবহারকারীর দশ লক্ষ request, সবকিছু একসাথে ঘটলে কী হবে? একটি একক server, যতই শক্তিশালী হোক না কেন, শেষ পর্যন্ত আর সামলাতে পারে না। এটাই scalability সমস্যা, এবং এটি সমাধানের ঠিক দুটি মৌলিক উপায় আছে: আপনার একটি server-কে বড় করা, অথবা আরও server যোগ করা। চলুন দুটোই খুঁটিয়ে দেখি।

### What is Scalability?

**Scalability** হলো একটি সিস্টেমের ক্রমবর্ধমান কাজের পরিমাণ — আরও ব্যবহারকারী, আরও request, আরও data — resource যোগ করে সামলানোর ক্ষমতা, আদর্শভাবে performance-এ কোনো পতন বা সিস্টেমের পুনর্লিখন ছাড়াই। এটি একটি non-functional requirement, যা আমরা video 2-তে পরিচয় করিয়েছিলাম, এবং আপনার প্রত্যাশিত scale জানার সাথে সাথেই এটি প্রথম যে বিষয়গুলো নিয়ে চিন্তা করা উচিত তার একটি।

গুরুত্বপূর্ণ বিষয় হলো, scalability শুধু আজকের load টিকিয়ে রাখা নয় — এটি ভবিষ্যতে সেই load-এর 10x বা 100x পরিমাণ কোনো মৌলিক redesign ছাড়াই সামলানোর একটি স্পষ্ট পথ থাকা। একটি সিস্টেম যা 1,000 ব্যবহারকারীতে ঠিকঠাক কাজ করে কিন্তু 10,000 ব্যবহারকারীতে সম্পূর্ণ ভেঙে পড়ে, সেটি scalable নয়, এমনকি এখন এটি একদম ঠিকঠাক দেখালেও।

### Vertical Scaling: Scaling Up

প্রথম কৌশলটি হলো **vertical scaling**, যাকে কখনো কখনো "scaling up" বলা হয়। এর অর্থ হলো আপনার *বিদ্যমান* machine-এ আরও শক্তি যোগ করা — আরও CPU core, আরও RAM, দ্রুততর storage (যেমন spinning hard disk থেকে SSD-তে upgrade করা)। নতুন server যোগ করার বদলে, আপনার হাতে থাকা একটি server-কেই আরও শক্তিশালী করা হয়।

এটাকে এভাবে ভাবুন — যখন আপনার আরও মালামাল বহন করতে হয়, তখন একটি compact car থেকে একটি truck-এ upgrade করার মতো — একই একটি vehicle, শুধু এর একটি বড় এবং আরও সক্ষম সংস্করণ।

Vertical scaling-এর প্রকৃত সুবিধা আছে: এটি সহজ। আপনার application code প্রায়শই একদমই পরিবর্তন করতে হয় না — একই একক server শুধু দ্রুত চলে বা আরও বেশি concurrent connection সামলায়। একাধিক machine সমন্বয় করার কোনো অতিরিক্ত জটিলতা নেই, তাদের মধ্যে যোগাযোগ কীভাবে হবে তা নিয়ে চিন্তা করার দরকার নেই, এবং তাদের মধ্যে data-consistency নিয়েও কোনো মাথাব্যথা নেই।

কিন্তু vertical scaling-এর একটি কঠিন সীমা (hard ceiling) আছে। একটি একক machine-এ কতটা CPU, RAM এবং storage থাকতে পারে তার একটি physical সর্বোচ্চ সীমা আছে — এবং একবার সেখানে পৌঁছালে, আপনি যতই টাকা খরচ করতে রাজি থাকুন না কেন, আপনি আটকে যাবেন। Cloud provider-রা খুব বড় machine বিক্রি করে ঠিকই, কিন্তু tier যত উপরে ওঠে, দাম তত অসামঞ্জস্যপূর্ণভাবে বেড়ে যায়, এবং শেষ পর্যন্ত বাজারের সবচেয়ে বড় machine-ও Google বা Netflix-এর মতো কোম্পানির জন্য যথেষ্ট বড় নয়। এখানে একটি reliability সমস্যাও আছে: একটি একক শক্তিশালী machine তবুও একটি **single point of failure**। এটি বন্ধ হয়ে গেলে, আপনার পুরো সিস্টেমও এর সাথে বন্ধ হয়ে যায় — এই ধারণাটি আমরা availability এবং fault tolerance নিয়ে পরবর্তী video-তে পুরোপুরি বিশ্লেষণ করব।

### Horizontal Scaling: Scaling Out

দ্বিতীয় কৌশলটি হলো **horizontal scaling**, বা "scaling out"। একটি machine-কে বড় করার বদলে, আপনি *আরও* machine যোগ করেন, এবং সব machine-এ কাজের চাপ ছড়িয়ে দেন। একটি truck-এর বদলে, এখন আপনার কাছে vans-এর একটি বহর (fleet) আছে, প্রতিটি load-এর একটি অংশ বহন করে।

Horizontal scaling-এর মূলত কোনো কঠিন সীমা নেই — আরও capacity দরকার? আরও machine যোগ করুন। Google এবং Amazon-এর মতো কোম্পানিগুলো আক্ষরিক অর্থে হাজার হাজার বা দশ হাজার হাজার পৃথক server জুড়ে service চালায়, যা একটি একক machine দিয়ে — তা যত বড়ই হোক না কেন — physically অসম্ভব হতো। এটি আরও resilient-ও বটে: একশোটি machine-এর একটি fleet-এ যদি একটি machine ব্যর্থ হয়, বাকি নিরানব্বইটি traffic serve করতে থাকে, এবং ব্যর্থ machine-টিকে কারো লক্ষ্য না করেই প্রতিস্থাপন করা যায়।

কিন্তু horizontal scaling এমন প্রকৃত জটিলতা নিয়ে আসে যা vertical scaling-এ নেই। প্রথমত, আপনার incoming request-গুলো এই সব machine-এ বিতরণ করার একটি উপায় দরকার — এটাই একটি **load balancer**-এর কাজ, যা আমরা Module 2-তে বিস্তারিত আলোচনা করব। দ্বিতীয়ত, আপনার application সাধারণত **stateless** হতে হবে, অর্থাৎ যেকোনো server আগের কোনো request থেকে শুধুমাত্র সেই একটি server-এর memory-তে সংরক্ষিত data-র উপর নির্ভর না করে যেকোনো request সামলাতে সক্ষম হওয়া উচিত — নাহলে, একজন ব্যবহারকারীর দ্বিতীয় request একটি ভিন্ন machine-এ পৌঁছাতে পারে যা তার প্রথম request সম্পর্কে কিছুই জানে না। তৃতীয়ত, আপনার server-গুলোর যদি data ভাগ করে নেওয়ার প্রয়োজন হয় — যেমন user session বা application state — তাহলে আপনার একটি shared data store (একটি database বা cache) প্রয়োজন যা সব server access করতে পারে, এবং একাধিক machine জুড়ে data সমন্বয় করা নিজস্ব চ্যালেঞ্জের একটি সেট তৈরি করে যা নিয়ে আমরা Module 3 এবং 6 জুড়ে যথেষ্ট সময় ব্যয় করব।

### Choosing Between Them

বাস্তবে, বেশিরভাগ real system-ই বিভিন্ন সময়ে **উভয়টিই** ব্যবহার করে। vertical scaling দিয়ে শুরু করা খুবই সাধারণ ব্যাপার, কারণ product-এর প্রাথমিক পর্যায়ে এটি সহজ এবং সস্তা — শুধু একটি বড় server নিন। কিন্তু চাহিদা বাড়ার সাথে সাথে, আপনি অবশ্যম্ভাবীভাবে সেই সীমায় পৌঁছাবেন, এবং প্রকৃত internet scale-এ পরিচালনার জন্য horizontal scaling অপরিহার্য হয়ে ওঠে। একটি কাজের নিয়ম (rule of thumb): vertical scaling আপনাকে সময় কিনে দেয়; horizontal scaling আপনাকে একটি ভবিষ্যৎ কিনে দেয়।

এখানে একটি cost dimension-ও আছে। একটি নির্দিষ্ট বিন্দুর পর, একটি একক machine-এর spec দ্বিগুণ করলে শুধু তার দামও দ্বিগুণ হয় না — এটি দ্বিগুণের চেয়ে অনেক বেশি খরচ করতে পারে, কারণ top-tier hardware-এ একটি premium থাকে। এদিকে, একটি mid-tier machine-এর দশগুণ শক্তিসম্পন্ন একটি machine-এর চেয়ে দশটি mid-tier machine প্রায়ই মোট খরচে সস্তা হতে পারে, এবং একই সাথে বোনাস হিসেবে redundancy-ও দেয়।

### Real-World Example

একটি ছোট startup-এর প্রথম কয়েক মাসের কথা ভাবুন। তারা হয়তো তাদের পুরো application — web server এবং database — একটি যুক্তিসঙ্গত আকারের cloud instance-এ চালায়। যখন তাদের ব্যবহারকারী সংখ্যা শত থেকে দশ হাজারে বৃদ্ধি পায়, তারা প্রথমে vertically scale করে সাড়া দেয়: আরও RAM এবং CPU সহ একটি বড় instance-এ upgrade করে। এটি কিছুদিনের জন্য কাজ করে। কিন্তু একবার তারা লক্ষ লক্ষ ব্যবহারকারীকে serve করা শুরু করলে, কোনো একক machine বাস্তবসম্মতভাবে সেই load সামলাতে বা গ্রহণযোগ্য redundancy দিতে পারে না, তাই তারা horizontal scaling-এ স্থানান্তরিত হয়: একটি load balancer-এর পেছনে একাধিক web server, প্রতিটি stateless, সবগুলো একটি shared, পৃথকভাবে scale করা database tier-এ পড়ে ও লেখে। এই একই pattern — প্রথমে vertical scaling, পরে horizontal scaling — শিল্পে সবচেয়ে সাধারণ বৃদ্ধির গল্পগুলোর একটি।

### Recap

সংক্ষেপে বলতে গেলে: scalability হলো resource যোগ করে ক্রমবর্ধমান load সামলানোর একটি সিস্টেমের ক্ষমতা। Vertical scaling মানে একটি machine-কে বড় করা — সহজ, কিন্তু একটি কঠিন সীমা এবং একটি single point of failure সহ। Horizontal scaling মানে আরও machine যোগ করা — কার্যত সীমাহীন এবং আরও resilient, কিন্তু load balancing, statelessness, এবং shared data store প্রয়োজন। বেশিরভাগ real system প্রথম দিকে vertical scaling ব্যবহার করে এবং বৃদ্ধির সাথে সাথে horizontal scaling-এ স্থানান্তরিত হয়।

### What's Next

আমরা *আরও* traffic সামলানোর জন্য scaling নিয়ে কথা বলেছি — কিন্তু *failure* সামলানোর ব্যাপারে কী? পরবর্তী video-তে, আমরা availability, reliability, redundancy, এবং fault tolerance নিয়ে আলোচনা করব: একটি সিস্টেম কতটা ভালোভাবে চালু থাকে তা বর্ণনা করার জন্য প্রয়োজনীয় শব্দভাণ্ডার, এমনকি যখন এর পৃথক অংশগুলো ব্যর্থ হয়। এটি আমরা যা মাত্র শিখলাম তার সাথে সরাসরি জড়িত, কারণ horizontal scaling — আরও machine যোগ করা — fault-tolerant সিস্টেম তৈরির জন্যও আমাদের হাতে থাকা সেরা tool-গুলোর একটি বলে প্রমাণিত হয়। সেখানে দেখা হচ্ছে।

## Key Takeaways

- Scalability হলো একটি সিস্টেমের ক্রমবর্ধমান load (ব্যবহারকারী, request, data) resource যোগ করে সামলানোর ক্ষমতা, কোনো মৌলিক redesign ছাড়াই।
- Vertical scaling (scaling up) = একটি বিদ্যমান machine-এ আরও শক্তি যোগ করা; সহজ কিন্তু একটি কঠিন physical/cost সীমা আছে এবং একটি single point of failure তৈরি করে।
- Horizontal scaling (scaling out) = আরও machine যোগ করা; কার্যত সীমাহীন এবং আরও resilient, কিন্তু load balancing, stateless application design, এবং shared data store প্রয়োজন।
- বেশিরভাগ সিস্টেম সরলতার জন্য vertical scaling দিয়ে শুরু করে এবং একটি একক machine-কে ছাড়িয়ে গেলে horizontal scaling-এ চলে যায়।
- Horizontal scaling হলো বড় আকারের সিস্টেমে scalability এবং fault tolerance উভয়ের ভিত্তি।
