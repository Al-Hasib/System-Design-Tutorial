# Study Notes: ACID vs BASE, Normalization vs Denormalization

## ACID Definitions

| অক্ষর | টার্ম | সংজ্ঞা |
|---|---|---|
| A | **Atomicity** | একটি transaction সম্পূর্ণভাবে ঘটবে অথবা একেবারেই ঘটবে না; partial write গুলো rollback করা হয়। |
| C | **Consistency** | একটি transaction database-কে এক valid state থেকে আরেক valid state-এ নিয়ে যায়, সব constraint, cascade, এবং trigger মেনে। |
| I | **Isolation** | সমান্তরাল transaction গুলো একে অপরের uncommitted intermediate state দেখতে পায় না। |
| D | **Durability** | একবার commit হয়ে গেলে, data crash এবং power loss-এর পরেও টিকে থাকে (সাধারণত একটি write-ahead log-এর মাধ্যমে)। |

> লক্ষ্য করুন: ACID-এ "Consistency" (প্রতি-transaction data validity) CAP-এর "Consistency" (সব নোড একই সময়ে একই data দেখে) থেকে ভিন্ন একটি ধারণা। এগুলো গুলিয়ে ফেলবেন না।

## BASE Definitions

| শব্দ | টার্ম | সংজ্ঞা |
|---|---|---|
| BA | **Basically Available** | সিস্টেম প্রতিটি request-এ একটি সাড়ার নিশ্চয়তা দেয়, এমনকি partial failure-এর সময়ও, block করার বদলে। |
| S | **Soft state** | সিস্টেমের state সময়ের সাথে পরিবর্তিত হতে পারে নতুন কোনো ইনপুট ছাড়াই, ব্যাকগ্রাউন্ড replication/convergence-এর কারণে। |
| E | **Eventual consistency** | যদি write বন্ধ হয়ে যায়, সব replica অবশেষে একই মানে converge করবে, কখন হবে তার কোনো নির্দিষ্ট সীমা ছাড়াই। |

## Isolation Levels (SQL Standard, দুর্বলতম থেকে শক্তিশালীতম)

1. **Read Uncommitted** — dirty read সম্ভব (অন্য transaction-এর uncommitted data দেখা যেতে পারে)।
2. **Read Committed** — কোনো dirty read নেই; অনেক database-এর ডিফল্ট (যেমন, PostgreSQL, Oracle)। Non-repeatable read সম্ভব।
3. **Repeatable Read** — একটি transaction-এর মধ্যে একই query একই row রিটার্ন করে; কিছু implementation-এ phantom row এখনও সম্ভব। MySQL/InnoDB-তে ডিফল্ট।
4. **Serializable** — সবচেয়ে কঠোর; transaction গুলো এমনভাবে আচরণ করে যেন sequentially execute হয়েছে। সর্বোচ্চ safety, সর্বনিম্ন concurrency/throughput।

সাধারণ নিয়ম: কঠোরতর isolation = কম anomaly কিন্তু বেশি locking/contention এবং কম throughput।

## ACID vs BASE Comparison

| দিক | ACID | BASE |
|---|---|---|
| Guarantee | শক্তিশালী transactional correctness | উচ্চ availability, আনুমানিক correctness |
| Consistency model | Strong / immediate consistency | Eventual consistency |
| CAP alignment | CP-leaning (availability-র চেয়ে consistency) | AP-leaning (consistency-র চেয়ে availability) |
| সাধারণ সিস্টেম | PostgreSQL, MySQL, Oracle, SQL Server | Cassandra, DynamoDB, Riak, MongoDB (default configs), CouchDB |
| Coordination খরচ | বেশি (lock, consensus, log) | কম (async replication) |
| Use case | Banking, ledger, inventory, order processing, টাকা/uniqueness constraint জড়িত যেকোনো কিছু | Social feed, view/like counter, activity log, caching layer, IoT telemetry |

## Normalization vs Denormalization

- **Normalization**: redundancy দূর করতে data-কে সম্পর্কিত table-এ সংগঠিত করা।
  - **1NF**: atomic column value (একটি field-এ repeating group/list নেই)।
  - **2NF**: প্রতিটি non-key column *সম্পূর্ণ* primary key-এর উপর নির্ভরশীল (composite key-এর ক্ষেত্রে প্রাসঙ্গিক)।
  - **3NF**: প্রতিটি non-key column *শুধুমাত্র* key-এর উপর নির্ভরশীল, অন্য কোনো non-key column-এর উপর নয় (transitive dependency দূর করে)।
  - লক্ষ্য: update/insert/delete anomaly এড়ানো (যেমন, এক হাজার row-এ নকল করা একটি ঠিকানা এডিট করা)।
- **Denormalization**: join এড়াতে এবং read দ্রুত করতে ইচ্ছাকৃতভাবে data নকল করা। NoSQL document model-এ সাধারণ (সম্পর্কিত data embed করা) এবং analytics/OLAP সিস্টেমে (wide, flattened table)।

| দিক | Normalization | Denormalization |
|---|---|---|
| Redundancy | সর্বনিম্ন (single source of truth) | বেশি (data রেকর্ড জুড়ে নকল করা) |
| Read speed | multi-entity read-এর জন্য ধীর (join প্রয়োজন) | দ্রুত (data একসাথে embed করা, কোনো join নেই) |
| Write speed | দ্রুত/সহজ (একবার লিখুন) | ধীর/জটিল (সব কপি আপডেট করতে হয়) |
| Storage | কম | বেশি |
| Consistency risk | কম (আপডেট করার একটাই জায়গা) | বেশি (কপি গুলো সিঙ্ক থেকে বিচ্যুত হতে পারে) |
| সাধারণ use case | OLTP সিস্টেম, financial/transactional data, relational schema | Read-heavy feed, dashboard, analytics, document database, caching |

## মূল তথ্য

- ACID প্রতি-transaction ভিত্তিতে সংজ্ঞায়িত; isolation স্পেকট্রাম (read uncommitted → serializable) ACID-এর *ভেতরে* একটি নব, আলাদা কোনো মডেল নয়।
- BASE ACID-এর একটি অনানুষ্ঠানিক counterpoint হিসেবে তৈরি করা হয়েছিল, যা বড় স্কেলের distributed/NoSQL সিস্টেমের প্রেক্ষাপটে জনপ্রিয় হয়েছিল (যেমন, Amazon Dynamo, Google Bigtable-যুগের সিস্টেম)।
- CP-leaning সিস্টেম (CAP theorem থেকে) সাধারণত ACID প্রদান করে; AP-leaning সিস্টেম সাধারণত BASE প্রদান করে — এই ম্যাপিং কোনো আইন নয়, বরং একটি শক্তিশালী প্রবণতা।
- Normalized schema এবং ACID transaction সাধারণত একসাথে চলে; denormalized schema এবং BASE/eventual consistency সাধারণত একসাথে চলে।

## Bullet Summary

- ACID = Atomicity, Consistency, Isolation, Durability — শক্তিশালী guarantee, CP-leaning, relational database।
- BASE = Basically Available, Soft state, Eventual consistency — দুর্বলতর guarantee, AP-leaning, distributed/NoSQL database।
- Isolation level ACID-এর ভেতরে correctness এবং concurrency-র মধ্যে trade-off টিউন করে।
- Normalization redundancy কমায় এবং write integrity রক্ষা করে; denormalization read দ্রুত করতে data নকল করে।
- ভুল/stale হওয়ার খরচের ভিত্তিতে বেছে নিন: বেশি খরচ → ACID + normalized; কম খরচ, scale/speed প্রয়োজন → BASE + denormalized।
