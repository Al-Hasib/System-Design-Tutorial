# অনুশীলন ও ইন্টারভিউ প্রশ্ন

**১. একটি monolith-এর প্রকৃত নির্ধারক বৈশিষ্ট্য কী — এটা কি "খারাপ কোড" নাকি অন্য কিছু?**
Deployment boundary। একটি monolith সংজ্ঞায়িত হয় এটি একটিমাত্র deployable unit হিসেবে তৈরি এবং ship করা হয় (সাধারণত একটি shared database সহ) বলে, এর অভ্যন্তরীণ code-এর মান বা সংগঠন দ্বারা নয়। একটি monolith অভ্যন্তরীণভাবে পরিচ্ছন্নভাবে modularized হতে পারে।

**২. microservices-এ স্থানান্তরের তিনটি বাস্তব খরচের নাম বলুন যা team-গুলো প্রায়ই কম মূল্যায়ন করে।**
- Distributed transaction বিনামূল্যের ACID transaction-এর জায়গা নেয় (saga বা 2PC দরকার)।
- Network call latency এবং partial-failure mode নিয়ে আসে যা in-process call-এর ছিল না।
- আপনার নতুন infrastructure দরকার: service discovery, distributed tracing/correlation ID, এবং প্রতি-service CI/CD ও monitoring।

**৩. একটি "modular monolith" কী এবং এটি কেন একটি ভালো ডিফল্ট শুরুর বিন্দু?**
এমন একটি monolith যার অভ্যন্তরীণ module-গুলোর সুস্পষ্ট boundary এবং interface আছে (এবং কখনো কখনো পৃথক schema) কিন্তু তবুও একটিমাত্র unit হিসেবে deploy হয়। এটি একটি distributed system-এর operational overhead ছাড়াই service separation-এর বেশিরভাগ maintainability সুবিধা দেয়, এবং ভবিষ্যতে microservices-এ বিভক্ত হওয়া সহজ করে তোলে কারণ seam গুলো ইতিমধ্যেই বিদ্যমান।

**৪. Strangler fig migration pattern বর্ণনা করুন।**
একটি risky big-bang প্রচেষ্টায় একটি monolith পুরোপুরি পুনর্লিখনের বদলে, আপনি এর সামনে একটি facade বা API gateway বসান এবং ধাপে ধাপে পৃথক পৃথক capability নতুন service-এ বের করেন, প্রতিটি নতুন service প্রস্তুত হলে সংশ্লিষ্ট traffic সেদিকে redirect করেন, যতক্ষণ না monolith "strangled" হয়ে প্রায় কিছুই না থাকে (বা একটি ছোট core থাকে)।

**৫. আপনি কখন একটি team-কে microservices সুপারিশ করবেন না?**
যখন team ছোট (মোটামুটি এক থেকে কয়েকটি ছোট team), domain boundary এখনও ভালোভাবে বোঝা যায়নি, শক্তিশালী cross-entity transactional consistency একটি ঘনঘন প্রয়োজনীয়তা, অথবা organization-এর distributed system operation সামলানোর মতো platform পরিপক্কতা (CI/CD, observability, on-call practice) নেই। এসব ক্ষেত্রে microservices-এর operational tax সুবিধার চেয়ে বেশি হয়ে যায়।

**৬. Conway's Law কী এবং এই সিদ্ধান্তের সাথে এটি কেন প্রাসঙ্গিক?**
Conway's Law বলে যে system গুলো প্রায়ই তাদের নির্মাণকারী organization-এর যোগাযোগ কাঠামোকে প্রতিফলিত করে। এটি প্রাসঙ্গিক কারণ microservices গ্রহণ প্রায়ই বিশুদ্ধ technical scaling-এর চেয়ে বেশি team autonomy (স্বাধীন deployment, ownership) সক্ষম করার বিষয়ে হয়ে থাকে — আপনি কাঙ্ক্ষিত team boundary-র সাথে মেলাতে service boundary ডিজাইন করছেন।

**৭. একটি interview-তে, আপনাকে একটি 6-জনের startup-এর জন্য একটি photo-sharing app ডিজাইন করতে বলা হয়েছে। Monolith নাকি microservices — এবং কেন?**
Monolith (আদর্শভাবে modular)। এই মাপে, একটি ছোট team স্বাধীন scaling বা team autonomy-র চেয়ে সরলতা, দ্রুত iteration, এবং কম operational overhead থেকে অনেক বেশি উপকৃত হয়, এবং এগুলোর কোনোটাই এখনো একটি বাস্তব বাধা নয়।

**৮. কী কারণে Segment তাদের কিছু microservices থেকে ফিরে এলো?**
তাদের microservices-গুলোর load এবং failure mode অত্যন্ত পরস্পর সম্পর্কিত ছিল, তাই fault-isolation এবং স্বাধীন-scaling সুবিধা বাস্তবে কখনোই সত্যিকারভাবে বাস্তবায়িত হয়নি, তবুও তারা শত শত service চালানোর পুরো দৈনিক operational খরচ — on-call বোঝা, deployment/coordination জটিলতা — বহন করছিল।

**৯. একটি monolith এবং একটি microservices architecture-এর মধ্যে fault isolation কীভাবে ভিন্ন?**
একটি monolith-এ, একটি module-এর একটি unhandled exception বা resource exhaustion (যেমন, memory leak) পুরো process-কে crash বা degrade করতে পারে, সব functionality বন্ধ করে দিতে পারে। Microservices-এ, একটি ব্যর্থ service আলাদা করা যায় — বিশেষত circuit breaker এবং bulkhead (Module 6)-এর সাহায্যে — যাতে অন্য service গুলো কাজ করতে থাকে, যদিও এর জন্য ইচ্ছাকৃত ডিজাইন দরকার; এটা স্বয়ংক্রিয় নয়।

**১০. একটি monolith-কে service-এ বিভক্ত করার সময় হয়েছে তার প্রমাণ হিসেবে আপনার কোন সংকেত খোঁজা উচিত?**
Team-এর বৃদ্ধি যা ক্রমাগত merge conflict এবং deployment সমন্বয় যন্ত্রণা সৃষ্টি করে; নির্দিষ্ট component (যেমন, video processing) যেগুলোর বাকি app-এর চেয়ে খুবই ভিন্ন scaling profile দরকার; availability-র জন্য স্বাধীন failure domain-এর প্রয়োজন; অথবা ভিন্ন team-এর system-এর নিজের অংশের মালিকানা এবং স্বাধীনভাবে release করার প্রয়োজন।

**১১. Microservices বেছে নেওয়া কি consistency guarantee সম্পূর্ণভাবে হারানোর সমার্থক?**
সম্পূর্ণভাবে নয়, তবে আপনি native cross-service ACID transaction হারাবেন। আপনাকে স্পষ্টভাবে Saga pattern (compensating transaction) বা, সংকীর্ণ ক্ষেত্রে, two-phase commit — উভয়ই Module 6-এ কভার করা হয়েছে — এর মতো pattern ব্যবহার করে consistency-র জন্য ডিজাইন করতে হবে, এবং প্রায়ই service-গুলোর মধ্যে eventual consistency গ্রহণ করতে হবে।

**১২. কেন "polyglot technology"-কে microservices-এর একটি সুবিধা হিসেবে উল্লেখ করা হয়, এবং এর trade-off কী?**
প্রতিটি service তার নির্দিষ্ট সমস্যার জন্য সবচেয়ে উপযুক্ত language, framework, বা database ব্যবহার করতে পারে (যেমন, একটি high-throughput service-এর জন্য Go, ML-এর জন্য Python)। Trade-off হলো বর্ধিত operational বৈচিত্র্য — আরও tooling, organization জুড়ে আরও skill দরকার, এবং team-গুলোর মধ্যে কম code/infra পুনর্ব্যবহার।
</content>
