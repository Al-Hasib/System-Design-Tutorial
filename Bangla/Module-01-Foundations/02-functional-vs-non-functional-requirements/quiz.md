# অনুশীলন ও Interview প্রশ্ন

১. **Functional requirements-এর সংজ্ঞা দিন এবং একটি ride-sharing app-এর জন্য দুটি উদাহরণ দিন।**
   Functional requirements বর্ণনা করে system কী করে। উদাহরণ: একজন rider একটি trip request করতে পারে; একজন driver একটি trip request accept বা decline করতে পারে।

২. **Non-functional requirements-এর সংজ্ঞা দিন এবং একই ride-sharing app-এর জন্য দুটি উদাহরণ দিন।**
   Non-functional requirements বর্ণনা করে system তার functions কতটা ভালোভাবে সম্পাদন করে। উদাহরণ: trip matching 5 সেকেন্ডের কম সময়ে সম্পন্ন হতে হবে; system-কে 99.9% সময় available থাকতে হবে।

৩. **Interviewer-রা কেন ইচ্ছাকৃতভাবে "design Twitter"-এর মতো অস্পষ্ট প্রম্পট দেয়?**
   দেখার জন্য যে candidate design করার আগে scale, read/write ratio, এবং consistency needs সম্পর্কে clarifying questions জিজ্ঞাসা করবে কিনা, ভিত্তিহীন ধারণা করার বদলে — এটি বাস্তব-জগতের product requirements-এর অস্পষ্টতাকে প্রতিফলিত করে।

৪. **পরিস্থিতি: আপনাকে একটি chat application design করতে বলা হয়েছে। Design করার আগে আপনি কোন তিনটি clarifying question জিজ্ঞাসা করবেন?**
   এই যেকোনো তিনটি: কতজন concurrent users/DAU? Message delivery কি real time-এ প্রয়োজন নাকি delay হতে পারে? আমাদের কি message history/persistence দরকার? Group chats নাকি শুধু 1:1? কী consistency guarantees দরকার (যেমন, message ordering)?

৫. **50 মিলিয়ন DAU-বিশিষ্ট একটি system-এর জন্য একটি back-of-the-envelope calculation করুন, যেখানে প্রতিটি user দিনে 20টি request করে।**
   50M × 20 = 1B requests/day। 1B / 86,400 ≈ গড়ে 11,600 RPS। Peak-এর জন্য ~2-3 গুণ করুন → peak-এ প্রায় 25,000-35,000 RPS।

৬. **Read-to-write ratio কেন একটি গুরুত্বপূর্ণ non-functional বিবেচ্য বিষয়?**
   এটি সরাসরি architecture-এর পছন্দকে প্রভাবিত করে — যেমন, একটি read-heavy system caching এবং read replicas থেকে ব্যাপকভাবে উপকৃত হয়, যেখানে একটি write-heavy system-এর দরকার দ্রুত, durable writes-এর জন্য optimized একটি design এবং সম্ভবত sharding।

৭. **একই functional requirement কিন্তু ভিন্ন non-functional requirements-বিশিষ্ট দুটি system-এর উদাহরণ দিন, যার ফলে ভিন্ন architecture তৈরি হয়েছে।**
   একটি personal to-do list app এবং একটি global banking ledger — উভয়েরই functional requirement হতে পারে "একটি transaction/task রেকর্ড করা", কিন্তু banking system-এর strict consistency এবং durability guarantees দরকার যা to-do app-এর দরকার নেই, যার ফলে সম্পূর্ণ ভিন্ন database এবং architecture-এর পছন্দ তৈরি হয়।

৮. **Non-functional requirement হিসেবে "durability" মানে কী, এবং এটি "availability" থেকে কীভাবে ভিন্ন?**
   Durability মানে হলো একবার data সফলভাবে save হয়ে গেলে, failure-এর সময়েও তা হারাবে না। Availability মানে হলো system একটি নির্দিষ্ট uptime rate-এ পৌঁছানো যায় এবং requests-এর জবাব দিতে পারে। একটি system available হতে পারে কিন্তু durable নয় (যেমন, শুধুমাত্র in-memory storage) অথবা durable হতে পারে কিন্তু সাময়িকভাবে unavailable।

৯. **পরিস্থিতি: একজন stakeholder বলে "আমাদের system-কে দ্রুত হতে হবে।" কোন follow-up প্রশ্ন এটাকে একটি কার্যকর non-functional requirement-এ রূপান্তরিত করে?**
   একটি নির্দিষ্ট target জিজ্ঞাসা করুন, যেমন, "সর্বোচ্চ গ্রহণযোগ্য response time কত — 100ms, 500ms, 2 সেকেন্ড? কোন operations-এর জন্য নির্দিষ্টভাবে?" অস্পষ্ট quality লক্ষ্যগুলোকে অবশ্যই পরিমাপযোগ্য threshold-এ রূপান্তরিত করতে হবে।

১০. **Capacity estimation-এ নির্ভুল হিসাবের বদলে মোটামুটি, order-of-magnitude সংখ্যা কেন ব্যবহার করা উচিত?**
    কারণ design-এর প্রথম দিকে, নির্ভুল সংখ্যা পাওয়া যায় না বা দরকারও হয় না — লক্ষ্য হলো system-এর একটি server দরকার নাকি হাজার হাজার তা চিহ্নিত করা, একটি নির্ভুল সংখ্যা গণনা করা নয়; মোটামুটি অনুমান বড় architecture সিদ্ধান্তকে দিকনির্দেশনা দেওয়ার জন্য যথেষ্ট।

১১. **Requirements gathering বাদ দিয়ে সরাসরি architecture-এ ঝাঁপিয়ে পড়ার একটি সম্ভাব্য পরিণতি কী?**
    আপনি এমন একটি system তৈরি করতে পারেন যা over-engineered (সময়/খরচ নষ্ট করে) বা under-engineered (প্রকৃত load বা reliability needs সামলাতে অক্ষম), এবং একটি interview-এর পরিবেশে, আপনি এমন কেউ হিসেবে দেখা যাওয়ার ঝুঁকিতে পড়েন যে কাজ করার আগে ধারণা যাচাই করে না।

১২. **প্রতিটিকে functional (F) বা non-functional (NF) হিসেবে শ্রেণীবদ্ধ করুন: (a) "Users তাদের password reset করতে পারবে," (b) "99.95% uptime," (c) "Search results 200ms-এর কম সময়ে ফিরে আসবে," (d) "Users তারিখ দিয়ে search results filter করতে পারবে।"**
    (a) F, (b) NF, (c) NF, (d) F।
</content>
