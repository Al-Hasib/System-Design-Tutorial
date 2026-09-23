# অনুশীলন ও ইন্টারভিউ প্রশ্ন

1. **Database replication কী, এবং এটি কোন দুটি প্রধান সমস্যা সমাধান করে?**
   Replication হলো একাধিক database node জুড়ে একই ডেটা কপি করা এবং বজায় রাখা। এটি সমাধান করে (১) availability/durability — একটি node ব্যর্থ হওয়া মানে ডেটা হারানো বা downtime নয়, কারণ অন্য একটি node-এর কাছে একটি কপি আছে — এবং (২) read scaling — read ট্রাফিক একটির বদলে একাধিক node জুড়ে ছড়িয়ে দেওয়া যায়।

2. **Synchronous এবং asynchronous replication-এর মধ্যে পার্থক্য কী?**
   Synchronous replication client-কে acknowledge করার আগে replica-র কাছ থেকে confirmation-এর জন্য অপেক্ষা করে যে সে একটি write পেয়েছে, latency-র বিনিময়ে durability-কে অগ্রাধিকার দিয়ে। Asynchronous replication সাথে সাথে client-কে acknowledge করে এবং ব্যাকগ্রাউন্ডে replicate করে, গতিকে অগ্রাধিকার দিয়ে কিন্তু এমন একটি window তৈরি করে যেখানে primary-র ডেটা তখনো replica-দের কাছে পৌঁছায়নি।

3. **Replication lag কী, এবং এটি সাধারণত কেন থাকে?**
   Replication lag হলো একটি write primary/master-এ পড়া এবং সেই write একটি follower/replica-তে দৃশ্যমান হওয়ার মধ্যকার বিলম্ব। এটি মূলত থাকে কারণ বেশিরভাগ সিস্টেম পারফরম্যান্সের জন্য asynchronous replication ব্যবহার করে, তাই replica-রা primary-র সামান্য পরে পরিবর্তনগুলো প্রয়োগ করে।

4. **পরিস্থিতি: একজন ব্যবহারকারী একটি মন্তব্য পোস্ট করেন, মন্তব্য পাতায় redirect হন, এবং তার নিজের মন্তব্যটি অনুপস্থিত। কী ঘটছে এবং আপনি কীভাবে এটি ঠিক করবেন?**
   এটি চিরায়ত read-your-writes problem: write master-এ গিয়েছিল, কিন্তু পাতার read এমন একটি replica থেকে সার্ভ করা হয়েছিল যা তখনো ধরতে পারেনি (replication lag)। সমাধানের মধ্যে রয়েছে সেই ব্যবহারকারীর তাৎক্ষণিক পরবর্তী read অল্প সময়ের জন্য master-এ পাঠানো, একটি replication position/token ট্র্যাক করা এবং replica সেটি ধরতে অপেক্ষা করা, অথবা write-এর পর master-এ session affinity ব্যবহার করা।

5. **Master-Slave replication-এ, read কেন write-এর চেয়ে অনেক সহজে scale করতে পারে?**
   কারণ আপনি যত খুশি follower/replica node যোগ করতে পারেন, প্রতিটি স্বাধীনভাবে read ট্রাফিক সার্ভ করতে সক্ষম, যেখানে সব write অবশ্যই একটিমাত্র master দিয়ে যেতে হবে — এই topology-তে আরও master যোগ করা সম্ভব নয়, তাই write throughput একটি মেশিন যা সামলাতে পারে তার দ্বারা সীমাবদ্ধ।

6. **Failover কী, এবং এটি কেন ঝুঁকিপূর্ণ?**
   Failover হলো একটি master/primary ব্যর্থ হয়েছে তা সনাক্ত করা এবং তার জায়গা নিতে একটি replica promote করা। এটি ঝুঁকিপূর্ণ কারণ সেখানে একটি সনাক্তকরণ ও promotion বিলম্ব থাকে (সেই window-এ write অনুপলব্ধ থাকে), এবং যদি replication asynchronous ছিল, তাহলে promote করা replica-তে তখনো না পৌঁছানো যেকোনো write হারিয়ে যেতে পারে।

7. **Master-Master (multi-leader) replication কোন সমস্যাটি সমাধান করে যা Master-Slave করতে পারে না?**
   এটি একাধিক node-কে সরাসরি এবং স্বাধীনভাবে write গ্রহণ করতে দেয়, node জুড়ে write scaling এবং multi-region active-active সেটআপ সক্ষম করে যেখানে ব্যবহারকারীরা low latency-সহ ভৌগোলিকভাবে কাছাকাছি একটি leader-এ write করে এবং একটি অঞ্চলের leader ব্যর্থ হলেও write availability অব্যাহত থাকে।

8. **Multi-leader replication-এ write conflict কী, এবং এটি সমাধানের দুটি কৌশলের নাম বলুন।**
   একটি write conflict ঘটে যখন দুটি leader প্রায় একই সময়ে একই রেকর্ডে একটি write গ্রহণ করে, একে অপরের কাছে replicate হওয়ার পর দুটি অসঙ্গত মান তৈরি করে। দুটি সমাধান কৌশল: last-write-wins (LWW), যা যে write-এর timestamp পরে সেটি রাখে, এবং vector clocks, যা প্রকৃত concurrency বনাম একটি write অন্যটির কার্যকারণে অনুসরণ করছে কিনা তা সনাক্ত করতে causal ইতিহাস ট্র্যাক করে (কখনো কখনো প্রকৃত conflict অ্যাপ্লিকেশনের কাছে প্রকাশ করে)।

9. **Last-write-wins (LWW) conflict resolution-এর একটি অসুবিধা কী?**
   এটি নীরবে একটি বৈধ, ইচ্ছাকৃত write বাতিল করে দিতে পারে শুধুমাত্র এই কারণে যে তার timestamp আগের ছিল — যার মধ্যে রয়েছে node জুড়ে সামান্য অসিঙ্ক্রোনাস clock-এর কারণে সৃষ্ট ক্ষেত্রগুলোও — এবং দুটি write মেলানোর জন্য অ্যাপ্লিকেশন বা ব্যবহারকারীর কোনো সুযোগ থাকে না।

10. **পরিস্থিতি: আপনি ১০০০:১ read-to-write অনুপাতসহ একটি read-heavy blog প্ল্যাটফর্ম ডিজাইন করছেন। আপনি কোন replication topology বেছে নেবেন, এবং কেন?**
    Master-Slave (leader-follower)। যেহেতু write বিরল এবং read প্রাধান্য পায়, তাই একটি একক master সহজেই write ভলিউম সামলায় এবং load balancer-এর পেছনে read replica যোগ করা read ক্ষমতাকে লিনিয়ারভাবে scale করে — master-master-এর অতিরিক্ত conflict-resolution জটিলতার কোনো প্রয়োজন নেই।

11. **পরিস্থিতি: আপনি একটি বৈশ্বিকভাবে বিতরণকৃত collaborative অ্যাপ তৈরি করছেন যেখানে US, Europe, এবং Asia-র ব্যবহারকারীদের সবার low-latency write এবং আঞ্চলিক outage-এর সময় অব্যাহত availability দরকার। কোন topology উপযুক্ত, এবং আপনাকে আরও কী ডিজাইন করতে হবে?**
    Master-Master (multi-leader), প্রতি অঞ্চলে একটি master সহ, যাতে ব্যবহারকারীরা low latency-সহ তাদের নিকটতম অঞ্চলে write করতে পারে এবং একটি অঞ্চল বন্ধ হয়ে গেলেও অ্যাপ write-এর জন্য উপলব্ধ থাকে। আপনাকে একটি conflict resolution কৌশলও (যেমন, vector clocks বা app-level merge logic) ডিজাইন করতে হবে যেহেতু অঞ্চল জুড়ে একই ডেটায় একযোগে write প্রত্যাশিত।

12. **Complexity-র দিক থেকে Master-Slave এবং Master-Master-এর তুলনা করুন এবং কেন তা ব্যাখ্যা করুন।**
    Master-Slave সহজ: সেখানে একজন writer থাকে, তাই conflicting write সনাক্ত বা সমাধান করার কখনো প্রয়োজন হয় না, এবং একমাত্র প্রকৃত অপারেশনাল জটিলতা হলো failover প্রক্রিয়া। Master-Master বেশি জটিল কারণ এর জন্য leader-দের মধ্যে bidirectional replication এবং একটি সক্রিয় conflict detection/resolution কৌশল দরকার, যেহেতু একাধিক node একযোগে conflicting write গ্রহণ করতে পারে।
