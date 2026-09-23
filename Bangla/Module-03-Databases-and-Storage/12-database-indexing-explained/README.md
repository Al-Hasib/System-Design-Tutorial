# Database Indexing Explained (B-Trees, Hash Indexes)

**কঠিনতার মাত্রা:** Intermediate

## শেখার লক্ষ্য (Learning Objectives)

এই ভিডিও শেষে আপনি নিচের বিষয়গুলো পারবেন:

- একটি database index আসলে কী এবং কেন এটি query-কে অনেক বেশি দ্রুত করে তোলে তা ব্যাখ্যা করতে।
- একটি B-Tree (B+Tree) index-এর গঠন ও আচরণ এবং কেন এটি বেশিরভাগ relational database-এ default choice, তা বর্ণনা করতে।
- একটি hash index-এর গঠন ও আচরণ এবং কখন এটি B-Tree-কে ছাড়িয়ে যায়, তা বর্ণনা করতে।
- অন্যান্য প্রচলিত index type (composite, covering, full-text) চিহ্নিত করতে এবং সেগুলো কোন সমস্যার সমাধান করে তা বুঝতে।
- কোন পরিস্থিতিতে একটি index যোগ করা উপকারের চেয়ে বেশি ক্ষতি করে তা চিনতে।

## স্ক্রিপ্ট (Script)

### Hook/Intro

কল্পনা করুন, আপনি ১০ কোটি (100 million) row-বিশিষ্ট একটি table-এ একটি query চালাচ্ছেন: `SELECT * FROM users WHERE email = 'someone@example.com'`। একটি database-এ, সেই query মাত্র ২ মিলিসেকেন্ডে ফলাফল দেয়। আরেকটি database-এ — একই data, একই hardware — এটি ৪০ সেকেন্ড সময় নেয় এবং আপনার CPU-কে পুরোপুরি ব্যস্ত করে ফেলে। পার্থক্যটা কোথায়? প্রায় প্রতিটি ক্ষেত্রেই উত্তরটা একটাই: একটি index।

আজ আমরা indexing-এর রহস্য উন্মোচন করব। আমরা দেখব একটি index আসলে কী, এটি রাখার প্রকৃত খরচ কত, এবং তারপর database indexing-এর দুই প্রধান কর্মী — B-Tree এবং hash index — নিয়ে গভীরভাবে আলোচনা করব, যাতে আপনি শুধু "এখানে একটি index যোগ করো" এটুকু নয়, বরং *কেন* এটি কাজ করে এবং *কখন* কোন type ব্যবহার করতে হবে সেটাও বুঝতে পারেন।

### একটি Index আসলে কী?

মূলত, একটি database index হলো একটি আলাদা data structure যা একটি বা একাধিক column-এর sorted বা অন্য কোনোভাবে সংগঠিত একটি copy সংরক্ষণ করে, সাথে disk-এ থাকা সম্পূর্ণ row-এর একটি pointer রাখে। যা খুঁজছেন তা পেতে প্রতিটি row স্ক্যান করার পরিবর্তে — যাকে database-এর ভাষায় "full table scan" বলা হয় — database প্রায় সরাসরি মিলে যাওয়া row-গুলোতে চলে যেতে পারে।

চিরাচরিত উপমাটি হলো একটি textbook-এর শেষে থাকা index। "mitochondria" শব্দটির প্রতিটি উল্লেখ খুঁজতে চাইলে, আপনি ৬০০ পাতা উল্টে প্রতিটি পাতা পড়বেন না। আপনি index-এ যাবেন, "mitochondria" খুঁজে পাবেন, আর সেটি আপনাকে বলে দেবে: পাতা ৪৫, ১১২, ৩৮০। সেই index বইটির চেয়ে ছোট, এটি বর্ণানুক্রমে সাজানো, এবং এটি আগে থেকেই তৈরি করা থাকে যাতে খোঁজাখুঁজি দ্রুত হয়। একটি database index একটি column-এর জন্য একই কাজ করে: এটি সেই column-এর value-গুলোর ওপর একটি সংক্ষিপ্ত, সাজানো structure, যা সংশ্লিষ্ট row-গুলোর অবস্থান নির্দেশ করে।

ঠিক যেমন last name অনুযায়ী সাজানো একটি phone book। যদি phone book অসাজানো থাকত, তাহলে কারও নম্বর খুঁজতে প্রতিটি entry পড়তে হতো। সাজানো থাকার কারণে, আপনি সরাসরি "S" অংশে চলে যেতে পারেন। সেই সাজানো থাকাটাই *হলো* index।

### Index-এর খরচ (The Cost of Indexes)

Index বিনামূল্যে পাওয়া যায় না, আর এই অংশটাই বেশিরভাগ মানুষ এড়িয়ে যায়। আপনি যে প্রতিটি index তৈরি করেন, তা অতিরিক্ত কাজ ও অতিরিক্ত storage তৈরি করে:

- **Write amplification** — একটি table-এর যেকোনো `INSERT`, `UPDATE`, বা `DELETE`-এর সময় সেই table-এর প্রতিটি index-কেও update করতে হয়। যদি আপনার পাঁচটি index থাকে, তাহলে row-তে একটি write মানে হতে পারে মোট ছয়টি write: একটি table-এ (যাকে প্রায়ই heap বা clustered structure বলা হয়) এবং পাঁচটি প্রতিটি index-কে সমন্বয়ে রাখার জন্য।
- **Storage** — একটি index হলো সম্পূর্ণ একটি অতিরিক্ত data structure। একটি বড় table-এ কয়েকটি index সহজেই আপনার storage-এর পরিমাণ দ্বিগুণ বা তিনগুণ করে ফেলতে পারে।
- **Index maintenance** — data insert ও delete হওয়ার সাথে সাথে B-Tree index-গুলোকে rebalance, split করতে হয়, এবং মাঝে মাঝে দক্ষতা বজায় রাখতে rebuild বা reorganize (fragmentation সামলাতে) করতে হয়। এটি database engine-এর নিজে থেকে সামলানো background কাজ, কিন্তু এটি একেবারে খরচ-বিহীন নয়।

তাই indexing মূলত একটি trade-off: read performance কেনার জন্য আপনি write performance এবং disk space খরচ করছেন। যে column-গুলোতে আপনি ঘন ঘন filter, join, বা sort করেন, তার জন্য এই trade-off সাধারণত মূল্যবান — কিন্তু প্রতিটি table-এর প্রতিটি column-এর জন্য নয়, যা নিয়ে আমরা পরে ফিরে আসব।

### B-Tree Index বিস্তারিত

B-Tree — আরও নির্দিষ্টভাবে বলতে গেলে এর variant, B+Tree — PostgreSQL, MySQL-এর InnoDB engine, SQL Server, Oracle, এবং আপনি যেসব relational database-এর সম্মুখীন হবেন তার বেশিরভাগেই default index structure।

গঠনগতভাবে, একটি B-Tree হলো একটি balanced, sorted tree। উপরে একটি root node থাকে, মাঝে internal node-গুলো থাকে যা signpost-এর মতো কাজ করে, এবং নিচে leaf node-গুলো থাকে যেখানে প্রকৃত indexed value থাকে। "Balanced" মানে হলো root থেকে leaf পর্যন্ত প্রতিটি path একই দৈর্ঘ্যের — কোনো এক-পাশে বেশি গভীর branch নেই যা অন্যগুলোর চেয়ে অনেক বেশি গভীর। এই ভারসাম্যই ধারাবাহিক, পূর্বানুমানযোগ্য performance নিশ্চিত করে: আপনি প্রথম row খুঁজুন বা এক কোটিতম row, আপনাকে প্রায় একই সংখ্যক level অতিক্রম করতে হয় — কোটি কোটি row-বিশিষ্ট table-এর জন্যও সাধারণত মাত্র তিন বা চারটি level। এটি lookup, insert, এবং delete-এর জন্য logarithmic time complexity, O(log n), দেয়।

বিশেষভাবে B+Tree-তে — বাস্তবে ব্যবহৃত variant — সব প্রকৃত data pointer leaf node-গুলোতে থাকে, এবং সেই leaf node-গুলো sorted order-এ, বাম থেকে ডানে, একটি chain-এ যুক্ত থাকে। এই বিষয়টি অত্যন্ত গুরুত্বপূর্ণ: এর মানে হলো একবার আপনার শুরুর point খুঁজে পেলে, আপনি leaf-গুলোর মধ্য দিয়ে পাশাপাশি হেঁটে value-এর একটি *range* দক্ষভাবে সংগ্রহ করতে পারেন। এ কারণেই B-Tree নিচের ক্ষেত্রে চমৎকার:

- সঠিক মিল খোঁজা (Exact match lookups): `WHERE user_id = 42`
- Range query: `WHERE created_at BETWEEN '2026-01-01' AND '2026-02-01'`
- Sorting: `ORDER BY last_name`
- Prefix মিল: `WHERE last_name LIKE 'Smith%'`

Postgres এবং MySQL InnoDB উভয়ই কারণবশত B+Tree index-কে default হিসেবে ব্যবহার করে: এটি সবচেয়ে সেরা general-purpose structure, যা equality, range, এবং ordering — সবকিছু একসাথে সামলায়।

### Hash Index বিস্তারিত

একটি hash index সম্পূর্ণ ভিন্ন পদ্ধতি অবলম্বন করে। sorted tree-এর পরিবর্তে, এটি indexed value-কে একটি hash function-এর মধ্য দিয়ে চালিয়ে একটি hash code গণনা করে, এবং সেই hash code ব্যবহার করে ঠিক করে value-এর pointer কোন "bucket"-এ যাবে। কিছু খুঁজতে, database আপনার search value-টিকে hash করে, সরাসরি সংশ্লিষ্ট bucket-এ চলে যায়, এবং pointer-টি নিয়ে নেয়। কোনো traversal নেই, একাধিক level জুড়ে কোনো comparison নেই।

এটি exact-match lookup-এর জন্য, table-এর আকার নির্বিশেষে, গড়ে O(1) — constant time — দেয়। শুধুমাত্র equality check-এর জন্য, `WHERE user_id = 42`, এটি একটি B-Tree-এর চেয়েও দ্রুত হতে পারে।

কিন্তু এখানেই সমস্যা, আর এটি একটি বড় সমস্যা: hashing order নষ্ট করে দেয়। "apple"-এর hash এবং "apricot"-এর hash-এর মধ্যে কোনো সম্পর্ক নেই, যদিও শব্দ দুটি বর্ণানুক্রমে কাছাকাছি। এর মানে হলো hash index range query, `ORDER BY`, বা prefix matching সমর্থন করতে **পারে না** — "কাছাকাছি" value-গুলোর মধ্য দিয়ে হাঁটার কোনো উপায় নেই কারণ কোনোকিছুই অর্থগতভাবে কারও কাছাকাছি সংরক্ষিত নেই। একটি hash index-কে "১০০ থেকে ২০০-এর মধ্যে সবকিছু" জিজ্ঞাসা করুন, এবং প্রতিটি entry পরীক্ষা করা ছাড়া তার আর কোনো ভালো উপায় নেই।

এই কারণেই hash index নির্দিষ্ট, লক্ষ্যভিত্তিক জায়গায় দেখা যায়: PostgreSQL শুধুমাত্র equality-based workload-এর জন্য একটি স্পষ্ট index type হিসেবে hash index দেয়, Redis তার অনেক data structure-এর জন্য অভ্যন্তরীণভাবে hash table ব্যবহার করে, এবং in-memory hash map (Python-এ dictionary, Java-তে `HashMap`) হলো database-এর বাইরে সম্পূর্ণভাবে classic general-purpose ব্যবহারের ক্ষেত্র। কিন্তু একটি relational table-এর *default* index হিসেবে, hash index B-Tree-এর কাছে হেরে যায় কারণ বাস্তব application-গুলো প্রায় সবসময় কোথাও না কোথাও range query এবং ordering প্রয়োজন করে।

### অন্যান্য Index Type, সংক্ষেপে

জানার মতো আরও কিছু:

- **Composite (multi-column) index** — একাধিক column জুড়ে তৈরি একটি index, যেমন `(last_name, first_name)`। Column-এর ক্রম খুব গুরুত্বপূর্ণ: এই index শুধুমাত্র `last_name`-এর ওপর filter করা query, বা `last_name AND first_name` একসাথে filter করা query-কে দক্ষভাবে সেবা দেয়, কিন্তু শুধুমাত্র `first_name`-এর ওপর filter করা কোনো query-কে দক্ষভাবে সেবা দিতে পারে না।
- **Covering index** — একটি index যাতে একটি query-এর প্রয়োজনীয় প্রতিটি column অন্তর্ভুক্ত থাকে, ফলে database কখনো underlying table স্পর্শ না করেই সরাসরি index থেকে query-এর উত্তর দিতে পারে। এটি একটি অতিরিক্ত lookup ধাপ এড়িয়ে যায় এবং read-heavy query-এর জন্য বিশাল performance-এর লাভ হতে পারে।
- **Full-text index** — বড় text field-এর মধ্যে word এবং phrase খোঁজার জন্য তৈরি বিশেষায়িত structure (প্রায়ই inverted index), যা product search বা document search-এর মতো feature-এ ব্যবহৃত হয়।

### কখন Over-Index করা উচিত নয়

উপরের সবকিছু বিবেচনা করে, নিচের ক্ষেত্রে index যোগ করা ভুল পদক্ষেপ:

- **Write-heavy table** — যদি একটি table মূলত insert এবং update দ্বারা প্রাধান্যপ্রাপ্ত হয় (যেমন logging বা event table), তাহলে প্রতিটি অতিরিক্ত index প্রতিটি write-কে ধীর করে দেয়। এখানে index সর্বনিম্ন রাখুন।
- **ছোট table** — যদি একটি table-এ মাত্র কয়েকশো বা কয়েক হাজার row থাকে, তাহলে একটি full scan ইতিমধ্যেই দ্রুত; index-এর overhead হয়তো নিজের খরচ পুষিয়ে দিতে পারবে না।
- **Low-cardinality column** — `is_active` (boolean)-এর মতো একটি column বা মাত্র তিনটি সম্ভাব্য value থাকা `status`-এর মতো column indexing থেকে খুব বেশি উপকার পায় না, কারণ index খুব বেশি search-কে সংকীর্ণ করতে পারে না — আপনার অর্ধেক row-ই একই value শেয়ার করতে পারে।

### বাস্তব-জগতের উদাহরণ (Real-World Example)

একটি production e-commerce app কল্পনা করুন। এখানে ৫ কোটি (50 million) row-বিশিষ্ট একটি `orders` table আছে, এবং customer support-এর একটি dashboard আছে যা চালায়: `SELECT * FROM orders WHERE customer_email = 'jane@example.com' ORDER BY created_at DESC`। Support agent-রা অভিযোগ করছে যে প্রতিটি customer lookup-এর জন্য dashboard লোড হতে ৮-১২ সেকেন্ড সময় নেয়, এবং লোডের অধীনে এটি database CPU-কে হঠাৎ বাড়িয়ে দিচ্ছে।

সেই query-তে `EXPLAIN` চালালে একটি sequential scan দেখা যায় — database সবগুলো ৫ কোটি row পড়ছে এবং memory-তে filter করছে কারণ `customer_email`-এর ওপর কোনো index নেই। সমাধান: `CREATE INDEX idx_orders_customer_email ON orders(customer_email, created_at DESC);` — একটি composite index যা filter এবং sort order উভয়কেই কভার করে।

এটি যোগ করার পরে, একই query ৮-১২ সেকেন্ড থেকে কমে ১০ মিলিসেকেন্ডের নিচে চলে আসে, কারণ database এখন পুরো table স্ক্যান করার পরিবর্তে B-Tree-এর মাধ্যমে সরাসরি মিলে যাওয়া row-গুলোতে চলে যায়, যা ইতিমধ্যেই `created_at DESC`-এর জন্য সঠিক সাজানো ক্রমে আছে। Trade-off হলো: প্রতিটি নতুন order insert এখন এই index-কেও update করে, এবং index নিজেই অতিরিক্ত disk space ব্যবহার করে — কিন্তু যে table-এ write-এর চেয়ে read অনেক বেশি হয় (support dashboard, order history page), সেখানে এই trade প্রতিবারই করার মতো।

### সারাংশ (Recap)

চলুন সবকিছু একত্র করি। একটি index হলো আপনার data-এর ওপর একটি আলাদা, সংগঠিত structure যা ধীর linear scan-কে দ্রুত, লক্ষ্যভিত্তিক lookup-এ পরিণত করে — ঠিক যেমন একটি বইয়ের শেষে থাকা index। সেই গতি বিনামূল্যে নয়: আপনাকে write overhead, storage, এবং maintenance-এর মূল্য দিতে হয়। B-Tree হলো balanced, sorted, general-purpose কর্মী, যা equality, range, এবং ordering-এর জন্য O(log n) performance দেয় — এ কারণেই এটি Postgres এবং MySQL-এ default। Hash index সেই নমনীয়তার বিনিময়ে raw গতি দেয়, exact-match lookup-এর জন্য O(1) দেয় কিন্তু range বা sorting-এর জন্য কিছুই দেয় না। আর এই দুটির বাইরে, composite index, covering index, এবং full-text index আরও বিশেষায়িত সমস্যার সমাধান করে। দক্ষতাটা হলো "সবকিছু index করো" নয় — বরং ঠিক কোন column-গুলো একটি index পাওয়ার যোগ্য এবং কোনগুলো নয়, তা জানা।

### পরবর্তীতে কী

Indexing একটি single database-এ দ্রুত data খুঁজে পাওয়ার সমস্যার সমাধান করে। কিন্তু যখন একটি database server যথেষ্ট নয় — যখন আপনার data-কে hardware failure থেকে বাঁচাতে হবে, বা একাধিক machine জুড়ে read scale করতে হবে — তখন কী হবে? এটাই আমরা পরবর্তী ভিডিওতে আলোচনা করব: **Database Replication: Master-Slave & Master-Master**। সেখানে দেখা হবে।

## মূল বিষয়বস্তু (Key Takeaways)

- একটি index হলো একটি আলাদা, সাজানো structure যা column value-কে row-এর অবস্থানের সাথে map করে, full table scan এড়িয়ে যায়।
- Index read performance-এর বিনিময়ে write performance এবং storage খরচ করে — প্রতিটি index প্রতিটি insert, update, এবং delete-এ overhead যোগ করে।
- B-Tree (B+Tree) index হলো balanced, sorted tree যা O(log n) performance দেয়; এগুলো Postgres এবং MySQL InnoDB-তে default কারণ এগুলো equality, range query, এবং sorting সামলায়।
- Hash index গড়ে O(1) exact-match lookup দেয় কিন্তু range query বা ordering সমর্থন করতে পারে না, কারণ hashing value-এর order নষ্ট করে দেয়।
- Composite index multi-column filter-কে দ্রুত করে (column order গুরুত্বপূর্ণ), covering index শুধুমাত্র index থেকেই query-এর উত্তর দিতে দেয়, এবং full-text index text search-এর সেবা দেয়।
- Write-heavy table, ছোট table, এবং low-cardinality column-এ over-indexing এড়িয়ে চলুন — overhead সুবিধার চেয়ে বেশি হতে পারে।
