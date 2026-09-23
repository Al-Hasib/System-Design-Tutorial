# Practice ও Interview প্রশ্নাবলী

1. **নিজের ভাষায় বলো, একটি relational (SQL) database কী?**
   একটি database যা row এবং column-এর fixed-schema table-এ data store করে, foreign key ব্যবহার করে সম্পর্কিত table-গুলোকে যুক্ত করে, এবং SQL ব্যবহার করে সেগুলো query ও join করতে দেয় — সাথে strong ACID transaction guarantee থাকে।

2. **NoSQL মানে কী, এবং এই নামটি কেন বিভ্রান্তিকর?**
   "Not only SQL।" এটা বিভ্রান্তিকর কারণ এটা একটি প্রযুক্তি বা query language নয় — এটা একটি ছাতা-শব্দ যা চারটি খুবই ভিন্ন model (document, key-value, wide-column, graph) কভার করে, যাদের প্রধান মিল হলো এগুলো traditional fixed-schema relational database নয়।

3. **NoSQL database-এর চারটি প্রধান category-র নাম দাও এবং প্রতিটির জন্য একটি করে উদাহরণ দাও।**
   Document (MongoDB), key-value (Redis অথবা DynamoDB), wide-column (Cassandra), graph (Neo4j)।

4. **ACID মানে কী, এবং সাধারণত এটি কোন ধরনের database-এর সাথে যুক্ত?**
   Atomicity, Consistency, Isolation, Durability — সাধারণত relational (SQL) database-এর সাথে যুক্ত, যা নিশ্চিত করে transaction হয় পুরোপুরি সফল নয়তো ব্যর্থ হয় এবং committed data ব্যর্থতার পরও টিকে থাকে।

5. **Normalization বনাম denormalization ব্যাখ্যা করো এবং কোন model কোনটি প্রাধান্য দেয়।**
   Normalization প্রতিটি fact একবার store করে এবং foreign key/JOIN দিয়ে record-গুলো যুক্ত করে (SQL-এর দ্বারা প্রাধান্য পাওয়া, duplication কমায়)। Denormalization যেভাবে পড়া হবে সেভাবে shape করা data duplicate করে, read-এর সময় join এড়িয়ে যায় (NoSQL-এর দ্বারা প্রাধান্য পাওয়া, read গতির বিনিময়ে storage/write জটিলতা গ্রহণ করে)।

6. **কেন অনেক NoSQL database traditional relational database-এর চেয়ে সহজে horizontally scale করে?**
   এগুলো সাধারণত core requirement হিসেবে cross-node JOIN বা strict multi-node transaction ছাড়াই design করা হয়, এবং এগুলো design অনুযায়ী node জুড়ে data partition (shard) করে, তাই আরও machine যোগ করলে সেই coordination overhead ছাড়াই capacity বাড়ে যা জটিল relational transaction এবং join node-গুলোর মধ্যে দাবি করত।

7. **"Eventual consistency" কী, এবং কেন একটি system strong consistency-র বদলে এটি বেছে নেবে?**
   একটি guarantee যে সব replica শেষপর্যন্ত একই value-তে converge করবে, কিন্তু একটি write-এর ঠিক পরের read stale data ফেরত দিতে পারে। System গুলো এটা বেছে নেয় higher availability এবং lower latency পাওয়ার জন্য, বিশেষত network partition-এর সময়, যখন তাৎক্ষণিক global consistency correctness-এর জন্য জরুরি নয়।

8. **Scenario: তুমি একটি social media app-এর user profile store এবং activity feed design করছো। কোন database type ব্যবহার করবে, এবং কেন?**
   User profile-এর জন্য একটি document store (যেমন, MongoDB) ভালো মানায় কারণ প্রতিটি profile একটি self-contained, nested object যার field প্রতিটি user-এর জন্য ভিন্ন হতে পারে। Activity feed-এর জন্য, একটি wide-column অথবা key-value store (যেমন, Cassandra অথবা Redis) মানায় কারণ feed গুলো high-volume, append-heavy, সময়ক্রম অনুযায়ী সাজানো, এবং একটি জানা access pattern দিয়ে পড়া হয় (user X-এর জন্য সাম্প্রতিক activity আনো) — ad-hoc join-এর দরকার হয় না।

9. **Scenario: তুমি একটি e-commerce platform-এর জন্য payment এবং order-processing backend তৈরি করছো। SQL নাকি NoSQL-এর দিকে ঝুঁকবে, এবং কেন?**
   SQL-এর দিকে ঝুঁকো — payment এবং order-এর জন্য strong consistency এবং atomic transaction দরকার (একটি payment হয় পুরোপুরি সফল হবে নয়তো পুরোপুরি ব্যর্থ হবে, inventory count নির্ভুল থাকতে হবে), এবং data স্বভাবতই relational (user, order, order item, product), যা relational database এবং ACID guarantee নিরাপদে সামলানোর জন্য তৈরি।

10. **Polyglot persistence কী, এবং বড় company গুলো কেন এটা ব্যবহার করে?**
    একটি system-এর মধ্যে একাধিক database type ব্যবহার করা, প্রতিটি সেই নির্দিষ্ট sub-problem-এর সাথে মেলানো যেটা সে সবচেয়ে ভালো সামলায় — যেমন, আর্থিক transaction-এর জন্য relational, caching/session-এর জন্য key-value, social connection-এর জন্য graph, event telemetry-র জন্য wide-column। Company গুলো এটা ব্যবহার করে কারণ একটি বড় system-এ প্রতিটি ধরনের data এবং access pattern-এর জন্য কোনো একক database model সর্বোত্তম নয়।

11. **কেন একটি "friends of friends" query-র জন্য একটি graph database একটি relational database-কে ছাড়িয়ে যেতে পারে?**
    একটি relational database-এ, সেই query-র জন্য একটি relationship table জুড়ে একাধিক self-join দরকার হয়, যা graph বড় হওয়ার সাথে সাথে ব্যয়বহুল হয়ে ওঠে। একটি graph database relationship-কে first-class edge হিসেবে store করে এবং সরাসরি node-থেকে-node traverse করতে পারে, যা multi-hop relationship query-কে নাটকীয়ভাবে দ্রুত করে তোলে।

12. **একজন junior engineer বলে "NoSQL মানেই SQL-এর নতুন, উন্নত version, তাই আমাদের সবসময় এটাই ব্যবহার করা উচিত।" তুমি কীভাবে উত্তর দেবে?**
    এটা একটা ভুল ধারণা — NoSQL কঠোরভাবে ভালো নয়, এটা trade-off-এর একটা ভিন্ন সেট (সহজ horizontal scaling এবং schema flexibility-র বিনিময়ে প্রায়ই কম consistency এবং query flexibility)। Structured, relationship-heavy data-র জন্য যেখানে strong consistency এবং জটিল ad-hoc query দরকার, সেখানে একটি relational database প্রায়ই এখনও সঠিক — এবং সহজতর — পছন্দ।
</content>
