# Notes: SQL vs NoSQL

## সংজ্ঞা

- **SQL (relational) database** — data-কে row এবং column দিয়ে তৈরি fixed-schema **table**-এ store করে, table-গুলোর মধ্যে relationship **foreign key**-র মাধ্যমে প্রকাশ করা হয় এবং query-র সময় **JOIN**-এর মাধ্যমে একত্রিত করা হয়। SQL (Structured Query Language) দিয়ে query করা হয়। উদাহরণ: PostgreSQL, MySQL, Oracle, SQL Server।
- **NoSQL database** — "not only SQL"; flexible অথবা কোনো fixed schema ছাড়া non-relational database-এর জন্য একটি ছাতা-শব্দ, সাধারণত অনেক machine জুড়ে horizontally scale করার জন্য design করা। একটি model নয় — চারটি প্রধান category (document, key-value, wide-column, graph)।

## SQL vs NoSQL তুলনা

| Dimension | SQL (Relational) | NoSQL |
|---|---|---|
| Schema | শুরুতেই fixed, বাধ্যতামূলক; পরিবর্তনের জন্য migration দরকার | Flexible/dynamic; field record-ভেদে ভিন্ন হতে পারে |
| Scaling | মূলত vertical (বড় machine); shard করা কঠিন | মূলত horizontal (আরও machine); sharding-এর জন্য তৈরি |
| Consistency | ডিফল্টভাবে strong consistency (ACID) | প্রায়ই eventual consistency (BASE); কিছু tunable consistency দেয় |
| Query language / flexibility | SQL; যেকোনো ad-hoc query এবং JOIN সমর্থন করে | Store-ভেদে ভিন্ন; সাধারণত সরল query, join নেই/সীমিত, জানা access pattern-এর জন্য denormalized data |
| Data relationships | Foreign key + JOIN-এর মাধ্যমে first-class | সাধারণত denormalized/embedded, অথবা (graph DB-র ক্ষেত্রে) first-class node/edge হিসেবে model করা |
| সবচেয়ে উপযুক্ত | Structured, relational data; integrity গুরুত্বপূর্ণ এমন transaction (banking, order, inventory) | High-scale, দ্রুত-পরিবর্তনশীল, বা loosely structured data; caching, catalog, social graph, event stream |
| উদাহরণ | PostgreSQL, MySQL, Oracle, SQL Server | MongoDB, Redis, DynamoDB, Cassandra, Neo4j |

## NoSQL Category-সমূহ

| Category | উদাহরণ | সাধারণ Use Case |
|---|---|---|
| Document | MongoDB, Couchbase | একটি সম্পূর্ণ জিনিস হিসেবে fetch করা nested, self-contained object — user profile, product catalog, CMS content |
| Key-value | Redis, DynamoDB, Memcached | সরল, অতি-দ্রুত lookup — caching, session, shopping cart |
| Wide-column | Cassandra, HBase | বিশাল write volume, time-series/event data, cluster জুড়ে range scan |
| Graph | Neo4j, Amazon Neptune | Relationship-নির্ভর data — social graph, recommendation engine, fraud detection |

## মূল তথ্য / Concept

- **ACID** (SQL default): Atomicity, Consistency, Isolation, Durability — transaction হয় পুরোপুরি সফল নয়তো পুরোপুরি ব্যর্থ হয়, এবং committed data crash-এর পরও টিকে থাকে।
- **BASE** (distributed NoSQL-এ সাধারণ): Basically Available, Soft state, Eventually consistent — তাৎক্ষণিক consistency-র চেয়ে availability এবং partition tolerance-কে প্রাধান্য দেয়।
- **Normalization** (SQL norm): প্রতিটি fact একবার store করো, foreign key দিয়ে reference করো — duplication এড়ায়, JOIN overhead-এর খরচ হয়।
- **Denormalization** (NoSQL norm): যেভাবে পড়া হবে সেভাবে shape করা data duplicate করো — জানা access pattern-এর জন্য দ্রুত read, copy সমন্বয় রাখার জন্য বেশি write-time/storage overhead।
- **Polyglot persistence**: একটি system-এ একাধিক database type ব্যবহার করা, প্রতিটি সেই sub-problem-এর সাথে মেলানো যেটার সাথে সবচেয়ে ভালো ফিট করে (যেমন, order/payment-এর জন্য relational, session/cache-এর জন্য key-value, social connection-এর জন্য graph, event telemetry-র জন্য wide-column)।
- সরাসরি **CAP theorem** (consistency, availability, partition tolerance trade-off) এবং horizontal বনাম vertical **scaling**-এর সাথে যুক্ত, দুটোই এই course-এর অন্যত্র আলোচনা করা হয়েছে।

## দ্রুত সারসংক্ষেপ

- SQL = structured, relational, strongly consistent, ডিফল্টভাবে vertically-scaled, flexible ad-hoc querying।
- NoSQL = flexible schema, ডিফল্টভাবে horizontally-scaled, প্রায়ই eventually consistent, জানা access pattern-এর জন্য optimized।
- চারটি NoSQL ধরন: document, key-value, wide-column, graph — অভ্যাস অনুযায়ী নয়, data shape অনুযায়ী বেছে নাও।
- পছন্দ নির্ভর করে: data shape, scale-এর চাহিদা, consistency-র চাহিদা, team-এর পরিচিতি/জানা query pattern-এর উপর।
- বেশিরভাগ বড় বাস্তব system একসাথে একাধিক database type ব্যবহার করে (polyglot persistence), শুধু একটি নয়।
</content>
