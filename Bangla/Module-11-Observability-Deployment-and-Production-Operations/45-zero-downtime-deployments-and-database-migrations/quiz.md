# অনুশীলন ও Interview প্রশ্ন

**১. পুরনো version বন্ধ করে নতুনটা চালু করলে কেন downtime হয়, এমনকি সংক্ষিপ্ত সময়ের জন্য হলেও?**
অনিবার্যভাবে একটা ফাঁক তৈরি হয় — যত ছোটই হোক না কেন — যেখানে কোনো request serve করার মতো instance available থাকে না, কারণ পুরনোগুলো বন্ধ হয়ে গেছে আর নতুনগুলো তখনও চালু হওয়া শেষ করেনি। বাস্তব scale-এ, খুব ছোট একটা ফাঁকও একটা দৃশ্যমান সংখ্যক চলমান request ব্যর্থ করে দেয়।

**২. একটা rolling deployment কীভাবে zero-instance ফাঁক এড়ায়?**
এটা একবারে অল্প কয়েকটা করে instance replace করে — একটা নতুন-version instance চালু করে, তার readiness probe pass করার জন্য অপেক্ষা করে, তারপর একটা পুরনো-version instance বন্ধ করে, এবং এটা পুনরাবৃত্তি করে — ফলে পুরো প্রক্রিয়া জুড়ে ready instance-এর মোট pool কখনো শূন্যে নামে না।

**৩. একটা rolling deployment-এর সময় কেন পুরনো ও নতুন application version-কে একে অপরের সাথে backward-compatible হতে হয়?**
কারণ rollout-এর পুরো সময় জুড়ে দুই version-ই একসাথে live traffic serve করে। নতুন version যদি একটা API response-এর গঠন বা schema element এমনভাবে পরিবর্তন করে যা পুরনো version handle করতে পারে না (বা উল্টোটা), তাহলে সেই overlap window-এ সত্যিকারের request ব্যর্থ হয়, শুধু তত্ত্বে নয়।

**৪. Rollback গতি এবং blast radius-এর দিক থেকে blue-green ও canary deployment তুলনা করো।**
Blue-green খুব দ্রুত rollback দেয় (শুধু router-কে অস্পৃষ্ট blue environment-এ ফিরিয়ে দাও) কিন্তু প্রতিটি cutover-এ blast radius হয় সব-অথবা-কিছুই না — router সুইচ হওয়ার মুহূর্তেই একটা খারাপ deploy ১০০% traffic-কে প্রভাবিত করে। Canary প্রথমে নতুন version-কে traffic-এর একটা ছোট percentage-এ ramp করে blast radius সীমিত রাখে, যাতে সবার উপর প্রভাব পড়ার আগেই সমস্যা ধরা পড়ে, কিন্তু এর rollback/ramp-down বেশি ক্রমান্বয়ে হয় এবং এগোনোর সিদ্ধান্ত নিতে চলমান metric-পর্যবেক্ষণ দরকার হয়।

**৫. একটা rolling deployment-এর তুলনায় blue-green deployment-এর প্রধান খরচ trade-off কী?**
Blue-green-এ বর্তমান environment-এর পাশাপাশি একটা সম্পূর্ণ, পুরোপুরি scale করা দ্বিতীয় environment চালাতে হয়, অন্তত সাময়িকভাবে — যা সুইচের সময় infrastructure খরচ প্রায় দ্বিগুণ করে দেয় — যেখানে একটা rolling deployment বিদ্যমান capacity পুনরায় ব্যবহার করে, দ্বিগুণ infrastructure ছাড়াই ধীরে ধীরে instance replace করে।

**৬. একটা rolling deployment-এর অধীনে একটা database column-এর নাম বদলে একই deploy-এ application-কে নতুন নামে সুইচ করা কেন বিপজ্জনক?**
Rollout-এর সময়, পুরনো-version pod-গুলো তখনও চলমান থাকে এবং column-টাকে তার আসল নামে query করতে থাকে। যদি সেই deploy সরাসরি column-এর নাম বদলে দেয়, সেই পুরনো-version pod-গুলো সাথে সাথেই ব্যর্থ হতে শুরু করবে কারণ তারা যে column-টা প্রত্যাশা করে সেটা আর অস্তিত্বে নেই — rollout window-এর সময় পুরনো ও নতুন version-কে নিরাপদে একসাথে থাকতে হওয়ার একটা প্রত্যক্ষ ফলাফল।

**৭. একটা database migration-এর জন্য expand-contract pattern-এর চারটা ধাপ বর্ণনা করো।**
Expand: পুরনোটাকে স্পর্শ না করেই নতুন schema element যোগ করো। Migrate: এমন application code deploy করো যা পুরনো ও নতুন — উভয় structure-এই write করে এবং পুরনো data backfill করে। Contract: পুরোপুরি rollout ও backfill হয়ে গেলে, পুরনো structure-এ write করা বন্ধ করো এবং read সম্পূর্ণভাবে নতুনটায় সুইচ করো। Cleanup: পরে, একটা আলাদা migration-এ, এখন-অব্যবহৃত পুরনো structure drop করো।

**৮. Expand-contract একটা atomic পরিবর্তনের বদলে কেন একাধিক আলাদা deploy হিসেবে করা হয়?**
কারণ rollout-এর সময় একসাথে চলমান পুরনো ও নতুন application instance-এর যেকোনো মিশ্রণের জন্যই প্রতিটি ধাপ নিরাপদ হতে হবে। ধাপগুলো একত্রিত করলে (যেমন, একই deploy-এ রিনেম করা এবং read সুইচ করা) — যে instance তখনও পুরনো structure প্রত্যাশা করা version চালাচ্ছে সেটা ভেঙে পড়বে, ঠিক যে failure mode এড়ানোর জন্যই expand-contract design করা হয়েছে।

**৯. পরিস্থিতি: আপনাকে একটা critical fix যত দ্রুত এবং নিরাপদে সম্ভব deploy করতে হবে, আপনি একটা খুব দ্রুত rollback option চান, এবং সাময়িকভাবে বেশি infrastructure খরচ সহ্য করতে পারেন। কোন deployment strategy সবচেয়ে উপযুক্ত, এবং কেন?**
Blue-green — এটা আপনাকে fix-টা একটা সম্পূর্ণ আলাদা environment-এ deploy করতে, সেটা যাচাই করতে, এবং atomically cut over করতে দেয়, এবং কিছু ভুল হলে প্রায়-তাৎক্ষণিক rollback available থাকে (router-কে তখনও চলমান পুরনো environment-এ ফিরিয়ে দিয়ে) — যা ঠিক গতি এবং নিরাপদ, দ্রুত rollback-কে খরচ-দক্ষতার চেয়ে অগ্রাধিকার দেওয়ার সাথে মিলে যায়।

**১০. সত্য নাকি মিথ্যা: একটা canary deployment নিশ্চয়তা দেয় যে একটা খারাপ নতুন version কখনোই কোনো বাস্তব ব্যবহারকারীকে প্রভাবিত করবে না।**
মিথ্যা। একটা canary deployment প্রথমে শুধু traffic-এর একটা ছোট percentage-এ নতুন version উন্মুক্ত করে blast radius সীমিত করে, কিন্তু নতুন version-এ সমস্যা থাকলে সেই percentage-এর বাস্তব ব্যবহারকারীরা তখনও প্রভাবিত হয় — লক্ষ্য হলো rollout ১০০% traffic-এ পৌঁছানোর আগেই সেটা ধরে ফেলা ও থামানো, কোনো ব্যবহারকারীর জন্যই সব ঝুঁকি দূর করা নয়।
