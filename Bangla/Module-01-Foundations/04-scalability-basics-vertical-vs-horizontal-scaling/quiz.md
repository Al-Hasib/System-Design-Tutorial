# Practice & Interview Questions

1. **Scalability-কে আপনার নিজের ভাষায় সংজ্ঞায়িত করুন।**
   Scalability হলো একটি সিস্টেমের ক্রমবর্ধমান load — আরও ব্যবহারকারী, request, বা data — resource যোগ করে সামলানোর ক্ষমতা, আদর্শভাবে কোনো performance পতন বা সম্পূর্ণ পুনর্লিখন ছাড়াই।

2. **Vertical scaling কী? একটি concrete উদাহরণ দিন।**
   Vertical scaling মানে একটি বিদ্যমান একক machine-এ আরও resource (CPU, RAM, storage) যোগ করা — উদাহরণস্বরূপ, আরও concurrent query সামলানোর জন্য একটি database server-কে 8GB থেকে 64GB RAM-এ upgrade করা।

3. **Horizontal scaling কী? একটি concrete উদাহরণ দিন।**
   Horizontal scaling মানে load সম্মিলিতভাবে সামলাতে আরও machine যোগ করা — উদাহরণস্বরূপ, একটি বড় server-এর বদলে একটি load balancer-এর পেছনে 10টি অভিন্ন web server instance চালানো।

4. **Vertical scaling-এর প্রধান সীমাবদ্ধতা কী?**
   একটি একক machine-কে কতটা upgrade করা যায় তার একটি কঠিন physical সীমা আছে, উঁচু প্রান্তে খরচ অসামঞ্জস্যপূর্ণভাবে বাড়ে, এবং এটি যতই শক্তিশালী হোক না কেন এটি একটি single point of failure থেকে যায়।

5. **Horizontal scaling সাধারণত এমন দুটি জিনিস প্রয়োজন যা vertical scaling-এর দরকার নেই?**
   Machine-গুলোতে traffic বিতরণ করার জন্য একটি load balancer, এবং stateless application design (প্রায়ই একটি shared external data store-এর সাথে যুক্ত) যাতে যেকোনো server যেকোনো request সামলাতে পারে।

6. **একটি একক, খুব শক্তিশালী server প্রয়োজনীয় load সামলাতে পারলেও কেন সেটি একটি reliability ঝুঁকি হিসেবে বিবেচিত হয়?**
   কারণ এটি একটি single point of failure — যদি সেই একটি machine crash করে বা maintenance প্রয়োজন হয়, তাহলে পুরো সিস্টেম অনুপলব্ধ হয়ে যায়, তার যত অতিরিক্ত capacity-ই থাকুক না কেন।

7. **Scenario: একটি startup-এর database server peak hour-এ 90% CPU utilization-এ চলছে, কিন্তু তারা আগামী বছর স্থির, মাঝারি বৃদ্ধি আশা করছে। একটি যুক্তিসঙ্গত স্বল্পমেয়াদী scaling প্রতিক্রিয়া কী, এবং কেন?**
   Vertical scaling (একটি বড় instance-এ upgrade করা) প্রায়ই স্বল্পমেয়াদে যুক্তিসঙ্গত — এটি সহজ, কোনো application পরিবর্তনের প্রয়োজন হয় না, এবং বৃদ্ধি অব্যাহত থাকলে দল আরও জটিল horizontal scaling-এর পরিকল্পনা করার সময় কিনে দেয়।

8. **Scenario: একটি viral launch-এর পর একটি কোম্পানির ব্যবহারকারী সংখ্যা 50x বেড়েছে, এবং একটি upgraded database server এখনো যথেষ্ট নয়। তাদের কী বিবেচনা করা উচিত, এবং এটি কী জটিলতা তৈরি করে?**
   তাদের horizontal scaling বিবেচনা করা উচিত — একাধিক machine-এ load ছড়িয়ে দেওয়া — যা load balancing, stateless service, এবং সেই machine-গুলো জুড়ে data ভাগ/সমন্বয় করার একটি strategy (যেমন replication বা sharding, Module 3-এ আলোচিত)-এর প্রয়োজনীয়তা তৈরি করে।

9. **একটি application server-এর "stateless" হওয়ার অর্থ কী, এবং horizontal scaling-এর জন্য এটি কেন গুরুত্বপূর্ণ?**
   একটি stateless server আগের কোনো request থেকে শুধু নিজের memory-তে সংরক্ষিত data-র উপর নির্ভর করে না — যেকোনো server যেকোনো request সামলাতে পারে। এটি গুরুত্বপূর্ণ কারণ একটি load balancer সময়ের সাথে একজন ব্যবহারকারীর request-গুলো বিভিন্ন server-এ route করতে পারে, এবং কারোরই সেই ব্যবহারকারীর আগের interaction সম্পর্কে ব্যক্তিগত, অ-ভাগাভাগি করা জ্ঞানের প্রয়োজন হওয়া উচিত নয়।

10. **কেন দশটি mid-tier server কখনো কখনো দশগুণ spec-এর একটি server-এর চেয়ে বেশি cost-effective হতে পারে?**
    High-end hardware প্রায়ই তার performance বৃদ্ধির তুলনায় একটি খাড়া price premium বহন করে (non-linear cost curve), তাই একই মোট capacity একাধিক mid-tier machine-এ বিতরণ করা সস্তা হতে পারে এবং একই সাথে redundancy-ও প্রদান করে।

11. **সত্য বা মিথ্যা: Horizontal scaling-এর capacity-র উপর কোনো ব্যবহারিক সীমা নেই।**
    নীতিগতভাবে মূলত সত্য — আপনি machine যোগ করতেই থাকতে পারেন — যদিও বাস্তবে আপনি অবশেষে নতুন bottleneck-এ পৌঁছাবেন (যেমন database contention, network সীমা, coordination overhead) যার জন্য sharding বা caching-এর মতো অন্য কৌশল প্রয়োজন, যা course-এ পরে আলোচিত হবে।

12. **ক্রমবর্ধমান startup-গুলোতে সাধারণ "vertical first, horizontal later" pattern-টি ব্যাখ্যা করুন।**
    শুরুতে, একটি কোম্পানি vertically scale করে কারণ এটি সহজ, দ্রুত, এবং কোনো architecture পরিবর্তনের প্রয়োজন হয় না — একটি বড় server-ই যথেষ্ট। বৃদ্ধি অব্যাহত থাকলে এবং একটি একক machine-এর সীমায় (capacity বা reliability) পৌঁছালে, তারা একাধিক server, একটি load balancer, এবং shared data store সহ horizontal scaling-এ স্থানান্তরিত হয়, যাতে অব্যাহত এবং আরও resilient বৃদ্ধি সমর্থন করা যায়।
