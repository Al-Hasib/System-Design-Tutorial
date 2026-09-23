# Practice & Interview Questions

**1. একটি cluster resize করার সময় naive `hash(key) mod N` partitioning কেন খারাপ পারফর্ম করে?**
কারণ modulus N প্রতিটি key-এর assignment formula-র অংশ, N পরিবর্তন করলে (node যোগ/অপসারণ) প্রায় প্রতিটি key-এর `mod N` result পরিবর্তিত হয়, পুরনো এবং নতুন assignment-এর মধ্যে কোনো structural সম্পর্ক ছাড়াই। সবচেয়ে খারাপ ক্ষেত্রে, একটি একক node পরিবর্তনের জন্য সব key-এর `(N-1)/N` পর্যন্ত remap হয়, যা ব্যাপক cache miss বা প্রায়-সম্পূর্ণ data reshuffle ঘটায়।

**2. আপনার নিজের ভাষায় hash ring মডেল বর্ণনা করুন।**
- Node এবং key উভয়কেই একই fixed range-এ (যেমন, 0 থেকে 2^32-1) hash করা হয়, যা একটি circle হিসেবে বিবেচিত হয় যা maximum value থেকে আবার শূন্যে wrap করে।
- প্রতিটি node তার hash অনুযায়ী ring-এ একটি position দখল করে।
- একটি key তার নিজের hashed position থেকে clockwise দিকে হেঁটে পাওয়া প্রথম node-এর অন্তর্গত।

**3. আপনার কাছে consistent hashing ব্যবহারকারী 5টি node-এর একটি cache cluster আছে এবং আপনি একটি 6ষ্ঠ node যোগ করেন। মোটামুটি কতগুলো key move করে, এবং কেন?**
মোটামুটি key-এর K/6 অংশ move করে (যেখানে K হলো মোট key সংখ্যা) — শুধুমাত্র ring-এর সেই arc-ই প্রভাবিত হয় যেখানে নতুন node নিজেকে insert করে, এবং সেই key-গুলো আসে ঠিক একটি বিদ্যমান প্রতিবেশী থেকে (যে node আগে সেই arc-এর মালিক ছিল)। বাকি সব node-এর key অস্পৃষ্ট থাকে, naive mod-N hashing-এর বিপরীতে, যা বেশিরভাগ key-ই remap করে দিত।

**4. Virtual node কোন সমস্যা সমাধান করে, এবং কীভাবে?**
প্রতি physical node-এ শুধু একটি hash position থাকলে, ring-এ random placement অত্যন্ত অসম arc length তৈরি করতে পারে, ফলে কিছু node অন্যদের চেয়ে অনেক বেশি keyspace-এর মালিক হয়ে যায়। Virtual node প্রতিটি physical node-কে একাধিকবার hash করে (সাধারণত ~100-200 position), প্রতি node-এ অনেকগুলো ছোট arc ring জুড়ে ছড়িয়ে দেয়; law of large numbers অনুযায়ী, প্রতিটি physical node-এর মোট মালিকানাধীন keyspace একটি সমান share-এর দিকে converge করে।

**5. Heterogeneous hardware capacity-র ক্ষেত্রে virtual node কীভাবে সাহায্য করে?**
আপনি একটি বেশি শক্তিশালী machine-কে বেশি সংখ্যক virtual node দিতে পারেন (যেমন, দ্বিগুণ RAM/disk-এর জন্য দ্বিগুণ vnode), ফলে এটি ring-এর arc-গুলোর সমানুপাতিকভাবে বেশি অংশের মালিক হয় এবং সেই অনুযায়ী সমানুপাতিকভাবে বেশি data ও traffic বহন করে — routing algorithm নিজে পরিবর্তন না করেই, শুধু প্রতি physical node-এ vnode সংখ্যার configuration পরিবর্তন করে।

**6. যখন একটি node অপ্রত্যাশিতভাবে ব্যর্থ হয়, তখন ring-এ, এবং data-তে কী ঘটে?**
ব্যর্থ node-এর virtual point ring থেকে সরানো হয় (বা অপৌঁছযোগ্য হিসেবে বিবেচিত হয়), এবং তার প্রতিটি arc সেই virtual point থেকে clockwise দিকে পরবর্তী node দখল করে নেয়। যেহেতু virtual node একটি physical node-এর মালিকানা অনেক ভিন্ন প্রতিবেশীর মধ্যে ছড়িয়ে দেয়, ব্যর্থ node-এর load একটি প্রতিবেশীর উপর সম্পূর্ণভাবে না পড়ে একাধিক physical node জুড়ে ছড়িয়ে পড়ে। Replicated সিস্টেমে, ring সেরে ওঠার সময় read/write বিদ্যমান replica-তে fail over করে।

**7. Consistent hashing-এর "consistent" কি CAP theorem-এর "consistency"-এর মতোই? ব্যাখ্যা করুন।**
না। Consistent hashing-এর "consistency" বোঝায় routing consistency: independent client বা node, ring membership-এর একই view দেওয়া থাকলে, কোনো coordination ছাড়াই একই key-এর জন্য একই owner গণনা করে। CAP-এর consistency বোঝায় একটি data-র সব replica একই, সবচেয়ে সাম্প্রতিক value প্রতিফলিত করে কিনা। একটি সিস্টেম hash ring-এর মাধ্যমে সম্পূর্ণভাবে consistent routing পেতে পারে অথচ CAP অর্থে একটি AP (eventually consistent) সিস্টেম হতে পারে — Dynamo এবং Cassandra এর ক্লাসিক উদাহরণ।

**8. প্রতিটি key-কে একটি একক node-এ সংরক্ষণ করার বদলে production সিস্টেমগুলো সাধারণত hash ring-এর উপরে replication কেন ব্যবহার করে?**
প্রতি key-তে একটি copy মানে যেকোনো একটি node-এর failure সেই key-এর arc-এর জন্য data loss বা unavailability ঘটাবে। Dynamo, DynamoDB, এবং Cassandra-র মতো সিস্টেমগুলো প্রতিটি key-কে তার position থেকে clockwise দিকে হেঁটে পরবর্তী R-টি ভিন্ন physical node-এ replicate করে (সাধারণত R=3), ফলে একটি node-এর failure মানে data হারানো নয় বরং replica-তে fall back করা, যার বিনিময়ে replica-গুলোর মধ্যে conflict resolution (vector clock, last-write-wins, read-repair) প্রয়োজন হয়।

**9. Topology পরিবর্তনের সময় একটি hash ring সব node এবং client জুড়ে কীভাবে synchronized থাকে?**
Ring membership প্রচার করা প্রয়োজন যাতে routing decision পুরো cluster জুড়ে converge করে। Cassandra-র মতো সিস্টেম gossip protocol ব্যবহার করে, যেখানে node-গুলো পর্যায়ক্রমে random peer-দের সাথে membership state বিনিময় করে যতক্ষণ না সবাই শেষ পর্যন্ত বর্তমান ring topology নিয়ে একমত হয়; topology পরিবর্তনের সময় সংক্ষিপ্ত সময়ের জন্য মতানৈক্য থাকতে পারে, যা replication এবং read-repair সামাল দিতে সাহায্য করে।

**10. একটি 10-node cluster-কে 11 node-এ scale করার সময় naive mod-N hashing এবং consistent hashing-এর rebalancing খরচ তুলনা করুন।**
- Naive mod-N: key-এর প্রায় (10)/(11) ≈ 91% পর্যন্ত remap হতে পারে, কারণ modulus নিজেই প্রায় সব key-এর জন্য পরিবর্তিত হয়।
- Consistent hashing: শুধু প্রায় 1/11 (~9%) key move করে — শুধু সেই arc যেখানে নতুন node নিজেকে insert করে, একটি বিদ্যমান প্রতিবেশী থেকে নেওয়া।
- এই order-of-magnitude পার্থক্যটিই মূল কারণ যে consistent hashing পছন্দ করা হয় এমন সিস্টেমের জন্য যাদের বড় data-transfer খরচ ছাড়াই incrementally scale করা প্রয়োজন।

**11. Consistent hashing ব্যবহারকারী দুটি বাস্তব-জগতের ধরনের সিস্টেম দিন, এবং প্রতিটির জন্য এটি কোন সমস্যা সমাধান করে।**
- Distributed NoSQL data store (DynamoDB, Cassandra, Riak): storage node জুড়ে data partition করা যাতে cluster প্রায় সব data move না করেই বাড়তে/ছোট হতে পারে, একই সাথে heterogeneous বা অসম hardware জুড়ে load balance করতে vnode ব্যবহার করা।
- CDN এবং load balancer: key (URL, client ID) অনুযায়ী request/cache lookup route করা যাতে একই key সব সময় একই backend-এ পৌঁছায় cache locality-র জন্য, একই সাথে পুরো routing/cache state একবারে অকার্যকর না করেই backend pool-এর পরিবর্তন সহ্য করা।

**12. ধরুন আপনার monitoring দেখাচ্ছে একটি consistent-hashing ring-এ একটি node সব সময় তার সহকর্মীদের দ্বিগুণ traffic সামলাচ্ছে, যদিও সব physical machine অভিন্ন। সম্ভাব্য কারণ এবং সমাধান কী?**
সম্ভাব্য কারণ হলো অপর্যাপ্ত সংখ্যক virtual node (বা একটি দুর্ভাগ্যজনক hash distribution) যার কারণে সেই physical node-এর মোট মালিকানাধীন arc length অসমানুপাতিকভাবে বড় হয়ে যাচ্ছে। সমাধান হলো প্রতি physical node-এ virtual node সংখ্যা বাড়ানো (~100-200+ পরিসরের দিকে) যাতে law of large numbers node জুড়ে arc-length distribution সমান করে দেয়; যদি hash function নিজেই খারাপ মানের হয়, তাহলে একটি ভালো-distribution-যুক্ত hash function-এ পরিবর্তন করাও সাহায্য করতে পারে।
