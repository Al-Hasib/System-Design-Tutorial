# স্টাডি নোটস: CAP Theorem ও PACELC

## সংজ্ঞা

- **Consistency (C)**: প্রতিটি read সবচেয়ে সাম্প্রতিক write ফেরত দেয় (অথবা একটি error)। সব node একই সময়ে একই ডেটা দেখে, যেন এর একটিমাত্র কপি আছে।
- **Availability (A)**: একটি non-failed node-এ প্রতিটি request একটি (non-error) response পায়, তবে তাতে সবচেয়ে সাম্প্রতিক write থাকার কোনো নিশ্চয়তা নেই।
- **Partition Tolerance (P)**: node-গুলোর মধ্যে network দিয়ে যতগুলো ইচ্ছা message drop বা delay হলেও, সিস্টেম কাজ চালিয়ে যায়।
- **PACELC**: CAP-এর extension — *যদি Partition থাকে, Availability বা Consistency বেছে নিন; Else (কোনো partition নেই, স্বাভাবিক কার্যক্রম), Latency বা Consistency বেছে নিন।*

## CAP Theorem Trade-off

| দিক | CP (Consistency + Partition Tolerance) | AP (Availability + Partition Tolerance) |
|---|---|---|
| Partition-এর সময় আচরণ | বাসি read বা হারানো write-এর ঝুঁকি না নিয়ে minority/unreachable পাশে request প্রত্যাখ্যান/block করে | উভয় পাশেই read/write সার্ভ করা চালিয়ে যায়, সম্ভাব্য বাসিভাব বা conflict মেনে নিয়ে |
| Consistency guarantee | Strong (linearizable/majority-based) | Eventual (পরে reconciliation-এর মাধ্যমে সমাধান করা হয়) |
| উদাহরণ database | ZooKeeper, etcd, HBase, MongoDB (majority read/write concern) | Cassandra, DynamoDB, Riak, CouchDB |
| সাধারণ ব্যবহারক্ষেত্র | Leader election, config/coordination service, financial ledger, oversell হওয়া উচিত নয় এমন inventory count | Shopping cart, social feed, session store, analytics/telemetry, product catalog |

দ্রষ্টব্য: একটি multi-node সিস্টেমের জন্য যা network-এর মাধ্যমে যোগাযোগ করে, "P ছাড়া C+A" কোনো বাস্তব বিকল্প নয় — partition জীবনের একটা বাস্তবতা, তাই বাস্তবে CAP আসলে একটি CP বনাম AP সিদ্ধান্ত।

## PACELC টেবিল

| সিস্টেম | P আচরণ (partition-এর সময়) | E আচরণ (স্বাভাবিক কার্যক্রম) |
|---|---|---|
| DynamoDB | PA — Availability-কে অগ্রাধিকার দেয় | EL — Latency-কে অগ্রাধিকার দেয় (default-এ eventually consistent read; উচ্চতর latency-তে ঐচ্ছিক strongly consistent read) |
| Cassandra | PA — Availability-কে অগ্রাধিকার দেয় (টিউনযোগ্য) | EL — default-এ Latency-কে অগ্রাধিকার দেয় (প্রতি query-তে consistency level টিউনযোগ্য: ONE/QUORUM/ALL) |
| MongoDB (majority read/write concern) | PC — Consistency-কে অগ্রাধিকার দেয় (majority-তে পৌঁছাতে না পারলে primary সরে দাঁড়ায়) | EC — Consistency-কে অগ্রাধিকার দেয় (majority acknowledgment-এর জন্য অপেক্ষা করে) |
| ZooKeeper / etcd | PC — Consistency-কে অগ্রাধিকার দেয় | EC — Consistency-কে অগ্রাধিকার দেয় (ডিজাইন অনুযায়ী linearizable read/write) |
| HBase | PC — Consistency-কে অগ্রাধিকার দেয় | EC — Consistency-কে অগ্রাধিকার দেয় (প্রতি region-এ একটি একক active region server) |

## মূল সূক্ষ্মতা (Key Nuances)

- CAP theorem **শুধুমাত্র একটি সক্রিয় network partition-এর সময়কার** আচরণ বর্ণনা করে — স্বাভাবিক, সুস্থ-network কার্যক্রমের সময়কার trade-off সম্পর্কে এটি কিছুই বলে না।
- একটি বাস্তব distributed system-এ partition tolerance ঐচ্ছিক নয়; এটি একটি প্রদত্ত বিষয়। অর্থবহ design পছন্দটি হলো কী ত্যাগ করবেন — Consistency নাকি Availability — *যখন* একটি partition ঘটে।
- একটি সিস্টেম কোনো পরিচয় হিসেবে স্থায়ীভাবে "CP" বা "AP" নয় — এটি একটি design পছন্দ যা বর্ণনা করে partition-এর মুহূর্তে সিস্টেমটি কীসকে অগ্রাধিকার দেয়; অনেক সিস্টেম প্রতি-request ভিত্তিতে টিউনযোগ্য (যেমন, Cassandra-র consistency level)।
- PACELC CAP-এর চেয়ে বেশি সম্পূর্ণ কারণ এটি সেই ধ্রুবক, প্রাত্যহিক latency-বনাম-consistency trade-offও ধারণ করে যা কোনো partition না থাকলেও বিদ্যমান থাকে।
- "সঠিক" trade-off একটি ব্যবসায়িক/domain সিদ্ধান্ত: ভুল/বাসি ডেটার খরচকে downtime বা ধীর response-এর খরচের সাথে তুলনা করুন।
- অনেক "AP" সিস্টেম চাহিদামতো এখনও শক্তিশালী consistency বিকল্প প্রদান করে (যেমন, DynamoDB-র strongly consistent read, Cassandra-র QUORUM) — লেবেলটি default/সাধারণ অবস্থান বর্ণনা করে, এটি কী করতে পারে তার একটি চূড়ান্ত সীমা নয়।
