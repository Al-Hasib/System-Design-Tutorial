# এই Topic কেন গুরুত্বপূর্ণ: SQL vs NoSQL

> **এক বাক্যে:** database হলো তোমার architecture-এর সেই জিনিস যা পরে পরিবর্তন করা সবচেয়ে কঠিন, আর access pattern-এর বদলে hype-এর ভিত্তিতে একটি বেছে নেওয়া design time-এ তোমার জন্য উপলব্ধ সবচেয়ে ব্যয়বহুল ভুল।

## এই Idea-র আগের জগৎ

প্রায় ত্রিশ বছর ধরে বেছে নেওয়ার কিছু ছিল না — তুমি একটি relational database ব্যবহার করতে, কারণ তখন সেটাই ছিল। এরপর internet scale-এর জন্য তৈরি system-গুলোর একটি ঢেউ এলো (Dynamo, BigTable, Cassandra, MongoDB, Redis), সাথে এলো একটি marketing narrative যে relational database "scale করে না।"

সেই narrative বিশাল ক্ষতি করেছিল। Team গুলো transactional, highly relational workload document store-এ migrate করেছিল, এরপর বছরের পর বছর application code-এ join পুনরায় implement করে কাটিয়েছিল, আবিষ্কার করেছিল যে তারা এমন transaction হারিয়েছে যা তাদের সত্যিই দরকার ছিল, এবং শেষমেশ আবার ফিরে migrate করেছিল। এদিকে অন্য team গুলো একটি সত্যিকারের key-value workload জোর করে Postgres-এ ঢুকিয়েছিল এবং ভাবছিল কেন সেটা ভেঙে পড়ল।

দুটো ব্যর্থতারই মূল কারণ একই: data এবং query-র shape দেখে নয়, বরং category দেখে database বেছে নেওয়া।

## এটি যে সমস্যাগুলো সমাধান করে

### ১. Application code-এ খারাপভাবে পুনরায় implement করা join
**তুমি যা দেখো:** একটি "customer এবং item সহ order আনো" endpoint ৪০টি আলাদা query চালায়, প্রতিটি একটি করে document ফেরত দেয়, যা application-এ একটি loop-এ জোড়া লাগানো হয়।

**কেন এটা ঘটে:** Document store ইচ্ছাকৃতভাবে join করে না। যদি তোমার data সত্যিকার অর্থে relational হয় — একাধিক দিকে একে অপরকে reference করা entity — তাহলে তোমার তখনও join দরকার; তুমি শুধু সেগুলো তোমার নিজের code-এ সরিয়ে নিয়েছ, যেখানে সেগুলো ধীর, unindexed, এবং untested।

**এই topic কীভাবে এটি সমাধান করে:** এটি তোমাকে প্রথমে *access pattern* দেখতে শেখায়। বহুভাবে interconnected data, যা অনেক combination-এ query করা হয়, সেটাই relational database এবং তাদের query planner-এর জন্য তৈরি করা হয়েছিল। এটা legacy নয়; এটা fit।

### ২. কারো সিদ্ধান্ত ছাড়াই হারিয়ে যাওয়া data integrity
**তুমি যা দেখো:** মুছে ফেলা product-কে reference করা order। `status: "activ"` সহ একটি user record। একই fact নিয়ে দ্বিমত পোষণ করা দুটো document।

**কেন এটা ঘটে:** Schema, foreign key, এবং constraint হলো enforcement, bureaucracy নয়। এগুলো সরিয়ে ফেললে system-এর প্রতিটি writer — বাগযুক্ত writer সহ, এবং কেউ ভোর ২টায় চালানো একবারের script-টিও — correctness-এর জন্য দায়ী হয়ে যায়।

**এই topic কীভাবে এটি সমাধান করে:** Schema-on-write বনাম schema-on-read একটি ইচ্ছাকৃত trade, বিনামূল্যের upgrade নয়। "Flexible schema" মানে "schema এখন তোমার application code-এ থাকে, data স্পর্শ করা প্রতিটি application-এ, চিরকালের জন্য।" কখনো কখনো এটাই সঠিক সিদ্ধান্ত। এটা একটা সিদ্ধান্ত হওয়া উচিত।

### ৩. একটি single node যা write volume সামলাতে পারে না
**তুমি যা দেখো:** তুমি প্রতি সেকেন্ডে দশ লক্ষ event ingest করছ। একটি primary node এটা শোষণ করতে পারে না, আর relational database-এ যোগ করা প্রতিটি shard operational কষ্ট বাড়ায়।

**কেন এটা ঘটে:** Classic relational database গুলো একটি single writable node এবং strong guarantee কেন্দ্র করে design করা হয়েছিল। Horizontal write scaling সম্ভব কিন্তু এটা একটা retrofit।

**NoSQL কীভাবে এটি সমাধান করে:** Cassandra এবং DynamoDB-এর মতো system মূল থেকেই partitioning কেন্দ্র করে design করা হয়েছিল — data key অনুযায়ী node-গুলোতে ছড়িয়ে থাকে, এবং node যোগ করলে write throughput প্রায় রৈখিকভাবে বাড়ে। Write-heavy, key-addressable workload-এর জন্য (time series, event, sensor data, session), এটি marketing নয়, একটি প্রকৃত architectural সুবিধা।

### ৪. তোমার দরকার নেই এমন guarantee-র জন্য মূল্য দেওয়া
**তুমি যা দেখো:** একটি session store বা computed result-এর cache তোমার প্রধান relational database-এ বসে থাকে, business-critical transaction-এর সাথে connection-এর জন্য প্রতিযোগিতা করে এবং পুরো system-কে টেনে নিচে নামায়।

**কেন এটা ঘটে:** সবকিছুর জন্য একটি database default হিসেবে ব্যবহার করা।

**এই topic কীভাবে এটি সমাধান করে:** Polyglot persistence — এই স্বীকৃতি যে ভিন্ন data-র ভিন্ন চাহিদা থাকে। Redis-এ session, Mongo-তে document, Postgres-এ transaction, Elasticsearch-এ search, Kafka-তে event। খরচ হলো পরিচালনা করার জন্য আরও বেশি system, তাই উত্তরটা "সবসময় আলাদা করো" নয় — কিন্তু "সবসময় একটাই database" ঠিক ততটাই চিন্তাহীন।

## যে মূল্য তোমাকে দিতে হবে

- **প্রতিটি অতিরিক্ত datastore একটি operational commitment।** Backup, monitoring, upgrade, expertise, on-call runbook। দুটো database একটির দ্বিগুণেরও বেশি কাজ মানে।
- **NoSQL তোমাকে আগে থেকেই query জানতে বলে।** Denormalized, query-first model তোমার design করা pattern-এর জন্য দ্রুত এবং যেগুলোর জন্য design করনি সেগুলোর জন্য প্রায় অসম্ভব। Requirement পরিবর্তিত হয়; data model প্রতিরোধ করে।
- **Family-র মধ্যে migration নিষ্ঠুর।** Postgres থেকে DynamoDB-তে পরিবর্তন করা driver swap নয়; এটা data model-এর এবং প্রায়ই application-এরও পুনর্গঠন।
- **আধুনিক database গুলো সীমারেখা ঝাপসা করে দেয়।** Postgres-এর strong JSON support আছে, MongoDB-এর transaction আছে, DynamoDB-এর secondary index আছে। পরিষ্কার dichotomy-টা তর্কে যতটা পরিষ্কার মনে হয় তার চেয়ে কম পরিষ্কার, যা category-র বদলে workload থেকে যুক্তি দেওয়ার আরেকটা কারণ।

## কখন এটা দরকার — এবং কখন নয়

| Relational-এর দিকে যাও যখন | NoSQL-এর দিকে যাও যখন |
|---|---|
| Data highly relational এবং অনেকভাবে query করা হয় | Access মূলত একটি জানা key দিয়ে হয় |
| তোমার multi-row ACID transaction দরকার | তোমার খুব উচ্চ write throughput এবং horizontal scale দরকার |
| Query pattern অনির্দেশ্যভাবে পরিবর্তিত হবে | Query pattern কম, জানা, এবং স্থিতিশীল |
| Integrity constraint গুরুত্বপূর্ণ (টাকা, inventory, identity) | Record-এর shape সত্যিই record-ভেদে ভিন্ন হয় |
| Scale "বড়" কিন্তু "internet-scale" নয় | Partition-এর সময় availability strict consistency-কে ছাড়িয়ে যায় |

**সৎ default:** relational দিয়ে শুরু করো যদি না তোমার একটি নির্দিষ্ট, স্পষ্টভাবে বলা কারণ থাকে অন্যথায় করার। Postgres মানুষ যা ভাবে তার চেয়ে অনেক বেশি load সামলাতে পারে, আর "হয়তো একদিন scale দরকার হবে" কোনো কারণ নয়।

## Interview-এ এটা কেন আসে

"তুমি কোন database ব্যবহার করবে এবং কেন?" প্রায় প্রতিটি design round-এ জিজ্ঞাসা করা হয়, আর ফাঁদটা হলো একটি product name দিয়ে উত্তর দেওয়া। প্রত্যাশিত উত্তর সিদ্ধান্তটাকে derive করে: এই যে data-র shape, এই যে read/write ratio, এই যে consistency requirement, তাই এই family, তাই এই product। যে candidate বলে "DynamoDB, কারণ আমাদের access pattern হলো খুব বেশি volume-এ user ID দিয়ে একটি single-key lookup এবং আমাদের cross-entity transaction দরকার নেই" — সে ভালো উত্তর দিয়েছে। "MongoDB, এটা বেশি scalable" — এটা ভালো উত্তর নয়।

## এটা কীভাবে সংযুক্ত

এই পছন্দ পরবর্তী অনেক কিছুকে চালিত করে। **Indexing** নির্ধারণ করে যাই বেছে নাও না কেন তার মধ্যে read performance। **Replication** এবং **sharding** হলো যেভাবে যেকোনো family scale করে। **CAP/PACELC** সেই consistency-availability trade-off ব্যাখ্যা করে যা NoSQL system গুলো স্পষ্ট করে তোলে। **ACID vs BASE** এবং **normalization vs denormalization** হলো এই সিদ্ধান্তের modeling পরিণতি। **LSM tree vs B-tree** ব্যাখ্যা করে কেন এই family গুলোর storage-engine স্তরে এত ভিন্ন write performance আছে।

**পরবর্তী:** [Database Indexing Explained](../12-database-indexing-explained/why.md) — যেকোনো ধরনের database কীভাবে তোমার data দ্রুত খুঁজে পায়।
</content>
