# অধ্যয়ন নোট: Database Sharding & Partitioning

## সংজ্ঞাসমূহ

- **Partitioning**: একটি dataset-কে ছোট, বেশি manageable অংশে ভাগ করা। একটি সাধারণ ছাতা-জাতীয় term।
- **Vertical partitioning**: ডেটাকে column/table/feature অনুযায়ী ভাগ করা (যেমন, `users` table একটি DB-তে, `orders` অন্য একটিতে; অথবা বড়/কম-ব্যবহৃত column আলাদা একটি table-এ ভাগ করা)। প্রতিটি partition-এর ভিন্ন schema/column-এর subset থাকে।
- **Horizontal partitioning (sharding)**: একটি single logical table-এর row-গুলো একাধিক স্বাধীন ডেটাবেস instance ("shard") জুড়ে ভাগ করা। প্রতিটি shard একই schema ভাগাভাগি করে কিন্তু ভিন্ন row-এর subset ধরে রাখে।
- **Shard key (partition key)**: যে column (বা column-এর সমন্বয়) দিয়ে নির্ধারণ করা হয় একটি নির্দিষ্ট row কোন shard-এর অন্তর্গত। একটি sharded সিস্টেমে সবচেয়ে গুরুত্বপূর্ণ design সিদ্ধান্ত।
- **Replication বনাম Sharding**: Replication একই *সম্পূর্ণ* dataset একাধিক node-এ কপি করে (availability + read scaling-এ সাহায্য করে)। Sharding *ভিন্ন* ডেটা একাধিক node জুড়ে ভাগ করে (write scaling + storage scaling-এ সাহায্য করে)। এগুলো একে অপরের বিকল্প নয়, বরং পরিপূরক — production সিস্টেমগুলো প্রায়ই shard করে ও প্রতিটি shard-কে replicate-ও করে।

## তুলনামূলক টেবিল: Sharding Strategies

| মানদণ্ড | Range-Based | Hash-Based | Directory-Based |
|---|---|---|---|
| বণ্টনের সমতা | দুর্বল–মাঝারি (key-এর distribution-এর উপর নির্ভর করে) | ভালো (ভালো hash fn দিয়ে প্রায় uniform) | ভালো (হাতে করে tune/rebalance করা যায়) |
| Range query সমর্থন | চমৎকার (contiguous range এক/কয়েকটি shard-এ map হয়) | দুর্বল (shard জুড়ে ছড়িয়ে থাকে; fan-out + merge দরকার) | মাঝারি (directory design-এর উপর নির্ভর করে) |
| Hotspot ঝুঁকি | বেশি (sequential/time-based key সবচেয়ে নতুন shard-এ load কেন্দ্রীভূত করে) | কম (pseudo-random বিস্তার) | কম–মাঝারি (হাতে rebalancing দিয়ে প্রশমিত) |
| Resharding-এর কঠিনতা | মাঝারি (range ভাগ করা যায়, কিন্তু data movement দরকার) | naive hash-mod-N দিয়ে বেশি (বেশিরভাগ key remap হয়); consistent hashing দিয়ে প্রশমিত | কম–মাঝারি (শুধু directory mapping আপডেট করলেই হয়) |
| Operational complexity | কম–মাঝারি | কম–মাঝারি | বেশি (তৈরি, scale, এবং available রাখার জন্য একটি অতিরিক্ত lookup service) |

## Cross-Shard Query ও Transaction চ্যালেঞ্জ

- **Join**: shard জুড়ে সম্পর্কিত ডেটা একটি single SQL query-তে join করা যায় না; একাধিক shard query করে এবং ফলাফল merge করে application code-এ join করতে হয়।
- **Transaction**: দুটি ভিন্ন shard-এর row জুড়ে বিস্তৃত একটি transaction local ACID guarantee-এর উপর নির্ভর করতে পারে না। এর জন্য distributed transaction pattern দরকার — two-phase commit (2PC) বা Saga pattern (compensating transaction) — উভয়ই latency এবং complexity যোগ করে।
- **Aggregation/analytics**: "পুরো dataset জুড়ে X-এর সাথে মিলে যাওয়া সব row গণনা করো"-এর মতো query-গুলোর জন্য query-কে প্রতিটি shard-এ ছড়িয়ে দিতে হয় এবং ফলাফল একত্র করতে হয় ("scatter-gather"), যা একটি single-shard query-এর চেয়ে ধীর এবং বেশি resource-নিবিড়।
- **Secondary index**: non-shard-key column-এর উপর একটি index সাধারণত একটি single shard-এ পরিষ্কারভাবে maintain করা যায় না; হয় index-টি shard জুড়ে duplicate করতে হবে (scatter-gather lookup), অথবা একটি আলাদা global index service maintain করতে হবে।
- **Rebalancing**: shard যোগ/বাদ দেওয়ার জন্য তাদের মধ্যে ডেটা সরাতে হয়। Naive hash-mod-N resize-এ প্রায় সব key remap করে; consistent hashing remapping-কে প্রায় `1/N` key-তে সীমাবদ্ধ রাখে।

## গুরুত্বপূর্ণ সংখ্যা / সাধারণ নিয়ম

- Naive `hash(key) % N` resharding: N-কে, ধরুন, ৪ থেকে ৫ shard-এ পরিবর্তন করলে প্রায় ৮০% key remap করার প্রয়োজন হতে পারে। Consistent hashing সাধারণত এটাকে প্রায় `1/N` key movement-এ সীমাবদ্ধ রাখে।
- Sharding বিবেচনা করার একটি সাধারণ trigger point: একটি single primary instance আর write throughput-এর সাথে তাল মেলাতে পারছে না, অথবা indexing এবং replication ইতিমধ্যে থাকা সত্ত্বেও working dataset আর comfortably memory/single disk-এ ধরছে না।
- ভবিষ্যতের split (অর্ধেক করা/দ্বিগুণ করা) সহজ করতে shard-সংখ্যা প্রায়ই দুই-এর power হিসেবে বেছে নেওয়া হয় (যেমন, ১৬, ৩২, ৬৪)।

## সংক্ষিপ্তসার পয়েন্ট

- Sharding write এবং storage scale করে; replication read এবং availability scale করে — বেশিরভাগ বড় সিস্টেমের উভয়ই দরকার।
- Vertical partitioning column/table অনুযায়ী ভাগ করে; horizontal partitioning (sharding) instance জুড়ে row অনুযায়ী ভাগ করে।
- Range sharding: সহজ range query, hotspot-প্রবণ।
- Hash sharding: সমান বণ্টন, দুর্বল range query।
- Directory-based sharding: সবচেয়ে flexible, অতিরিক্ত hop + infra খরচ।
- Consistent hashing resharding-এর সময় data movement কমিয়ে দেয় (deep dive আলাদাভাবে কভার করা হয়েছে)।
- উচ্চ cardinality, প্রধান query pattern-এর সাথে সামঞ্জস্য, এবং কম hotspot ঝুঁকি সহ একটি shard key বেছে নিন।
- Sharding-এর সবচেয়ে বড় খরচ হলো cross-shard join/transaction, rebalancing-এর পরিশ্রম, এবং বহুগুণ operational overhead।
