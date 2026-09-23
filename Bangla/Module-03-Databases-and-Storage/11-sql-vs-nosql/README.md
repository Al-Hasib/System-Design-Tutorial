# SQL vs NoSQL: সঠিক Database বেছে নেওয়া

**Difficulty:** Beginner/Intermediate

## Learning Objectives

- একটি relational (SQL) database কী এবং কীভাবে table, schema, row, এবং relationship একসাথে কাজ করে তা ব্যাখ্যা করা।
- একটি NoSQL database কী এবং এর চারটি প্রধান category — document, key-value, wide-column, এবং graph — বর্ণনা করা।
- schema flexibility, horizontal scaling, consistency বনাম availability, এবং query flexibility-এর ভিত্তিতে SQL এবং NoSQL-এর তুলনা করা।
- একটি নির্দিষ্ট system requirement-এর জন্য SQL বা NoSQL বেছে নিতে একটি বাস্তবসম্মত decision framework প্রয়োগ করা।
- "polyglot persistence" চিহ্নিত করা — একটি বাস্তব-জগতের system-এ একসাথে একাধিক database type ব্যবহার করা।

## Script

### Hook / Intro

সবাইকে স্বাগতম, আবার ফিরে আসার জন্য ধন্যবাদ। যদি তুমি foundations এবং scalability ভিডিওগুলো দেখে থাকো, তাহলে ইতিমধ্যেই জানো যে "আমরা আমাদের data কীভাবে store করব" — এটাই যেকোনো system design আলোচনার একদম প্রথম প্রশ্নগুলোর একটি। আর সেই পথের প্রথম fork প্রায় সবসময়ই: SQL নাকি NoSQL?

এই প্রশ্নটি প্রায় প্রতিটি system design interview-তে আসে, এবং এটি মানুষকে বিভ্রান্ত করে concept গুলো কঠিন বলে নয়, বরং বেশিরভাগ ব্যাখ্যা হয় খুব বেশি academic, নয়তো খুব বেশি ভাসাভাসা। তাই আজ আমরা সেটাই ঠিক করব। এই ভিডিওর শেষে, তুমি ঠিক জানবে SQL এবং NoSQL database আসলে কী, NoSQL-এর চারটি ধরন যা তোমার জানা দরকার, এদের মধ্যে আসল trade-off গুলো কী, এবং সঠিকটি বেছে নেওয়ার একটি সহজ framework — সেই সাথে একটি বাস্তব-জগতের উদাহরণ যেখানে একটি কোম্পানি একসাথে দুটোই ব্যবহার করছে।

### একটি SQL (Relational) Database কী?

চলো SQL দিয়ে শুরু করি, কারণ এটি দশকের পর দশক ধরে default হয়ে আছে, এবং এটি সেই mental model যার সম্পর্কে আমাদের বেশিরভাগেরই কিছুটা intuition আছে, এমনকি যদি আমরা এটা নিয়ে সরাসরি কখনো চিন্তা না করে থাকি।

একটি relational database data-কে **table**-এ organize করে — একটি table-কে spreadsheet-এর মতো ভাবো। প্রতিটি table-এর একটি নির্দিষ্ট **schema** থাকে: নির্দিষ্ট data type সহ column-এর একটি পূর্বনির্ধারিত সেট। একটি `users` table-এ `id` (একটি সংখ্যা), `name` (text), এবং `email` (text)-এর মতো column থাকতে পারে। সেই table-এর প্রতিটি **row** একটি record — একজন user — এবং এটি অবশ্যই সেই schema-এর সাথে মিলতে হবে। খুশিমতো একটি অতিরিক্ত field যোগ করা যায় না; নতুন column চাইলে সবার জন্য table structure পরিবর্তন করতে হয়।

relational model-এর আসল শক্তি হলো **relationship**। একজন user-এর তথ্য তার প্রতিটি order-এর মধ্যে বারবার duplicate না করে, তুমি user-দের একটি `users` table-এ store করো এবং order-গুলো আলাদা একটি `orders` table-এ, আর orders table শুধু একটি `user_id` রাখে — একটি **foreign key** যা `users`-এর সঠিক row-এর দিকে নির্দেশ করে। যখন পুরো ছবিটা দরকার হয়, তুমি **JOIN** ব্যবহার করে এই table গুলোকে তাৎক্ষণিকভাবে একসাথে জোড়া লাগাও। এতে duplication এড়ানো যায় এবং data consistent থাকে — user-এর email একবার update করলে, তাকে reference করা প্রতিটি order স্বয়ংক্রিয়ভাবে বর্তমান data প্রতিফলিত করে।

একটি relational database-এর সাথে তুমি **SQL** — Structured Query Language ব্যবহার করে কথা বলো। এটি declarative: তুমি বলো তোমার *কী* দরকার ("গত ৩০ দিনে দেওয়া $100-এর বেশি সব order আমাকে দাও, customer-এর নামসহ join করে") এবং database নিজেই বের করে নেয় দক্ষতার সাথে সেটা *কীভাবে* পেতে হবে।

সবশেষে, relational database গুলো **ACID** guarantee-এর জন্য পরিচিত — Atomicity, Consistency, Isolation, Durability। সহজ কথায়: একটি transaction হয় পুরোপুরি ঘটে, নয়তো একেবারেই ঘটে না (কোনো অর্ধেক-সম্পন্ন bank transfer নেই), database সবসময় এক valid state থেকে আরেক valid state-এ যায়, একইসাথে চলা transaction গুলো একে অপরের কাজে বাধা দেয় না, এবং একবার কিছু commit হয়ে গেলে, তা crash-এর পরও টিকে থাকে। Postgres, MySQL, Oracle, SQL Server-এর কথা ভাবো — এটাই সেই জগৎ যা তুমি ইতিমধ্যে চেনো।

### একটি NoSQL Database কী?

NoSQL — আক্ষরিক অর্থে "not only SQL" — আসলে সেই database গুলোর জন্য একটি ছাতা-শব্দ (umbrella term) যারা fixed-table-with-relationships model বাদ দিয়ে আরও flexible কিছু বেছে নেয়, এবং সাধারণত এমন কিছু যা মূল থেকেই অনেক machine জুড়ে horizontally scale করার জন্য তৈরি। একটিমাত্র NoSQL model নেই; চারটি প্রধান category আছে, এবং সঠিকটি বেছে নেওয়া ঠিক ততটাই গুরুত্বপূর্ণ যতটা প্রথমেই SQL বনাম NoSQL বেছে নেওয়া।

**১. Document store** — যেমন MongoDB। table-এর row-এর বদলে, তুমি পুরো JSON-এর মতো document store করো। একজন user এবং তার সব preference, address, এবং সাম্প্রতিক activity একটি একক document-এ থাকতে পারে, document-গুলোর মধ্যে কোনো schema বাধ্যতামূলক নয়। এটা দারুণ কাজ করে যখন তোমার data স্বাভাবিকভাবেই একটি nested object-এর মতো দেখায় এবং তুমি সাধারণত এটি একটি সম্পূর্ণ জিনিস হিসেবে fetch করো।

**২. Key-value store** — যেমন Redis বা DynamoDB। সবচেয়ে সরল model: তুমি একটি key দিয়ে একটি value খুঁজে বের করো, ব্যস। কোনো query নেই, কোনো join নেই, শুধু বিদ্যুৎ-গতির get-and-set। caching, session storage, এবং shopping cart-এর জন্য একদম উপযুক্ত, যেখানে তুমি সবসময় জানো ঠিক কোন key খুঁজছো।

**৩. Wide-column store** — যেমন Cassandra (এবং HBase)। এমন একটি table কল্পনা করো যেখানে প্রতিটি row-এ ভিন্ন এবং বিপুল সংখ্যক column থাকতে পারে, এবং data এমনভাবে physically organize করা থাকে যাতে একটি cluster জুড়ে বিশাল পরিমাণ write এবং range scan অত্যন্ত দ্রুত হয়। বিপুল পরিমাণ time-series বা event data ingest করা system-গুলোর পেছনের মূল শক্তি এটাই।

**৪. Graph database** — যেমন Neo4j। data node এবং edge হিসেবে store করা হয় — entity এবং তাদের মধ্যেকার relationship — যা connection traverse করার জন্য optimize করা, যেমন "friends of friends" বা "একসাথে প্রায়ই কেনা হয় এমন product"। যখন তোমার মূল প্রশ্নটা entity নিয়ে না হয়ে relationship নিয়ে হয়, তখন একটি graph database একটি relational JOIN chain-কে অনেক পেছনে ফেলে দেবে।

### মূল পার্থক্যসমূহ

চলো এগুলোকে সেই dimension গুলোর ভিত্তিতে পাশাপাশি সাজাই যা একটি system design করার সময় সত্যিই গুরুত্বপূর্ণ।

**Schema flexibility।** SQL শুরুতেই একটি কঠোর schema বাধ্যতামূলক করে — data integrity-র জন্য ভালো, কিন্তু প্রতিটি পরিবর্তনের জন্য একটি migration দরকার। NoSQL (বিশেষত document এবং wide-column store) তোমাকে প্রতিটি record-এ সাথে সাথে field যোগ করতে দেয়, যা দারুণ যখন তোমার data-র shape দ্রুত পরিবর্তিত হয় বা record-ভেদে ভিন্ন হয়।

**Horizontal scaling।** এটা বিশাল ব্যাপার। Relational database গুলো ঐতিহ্যগতভাবে *vertically* scale করার জন্য তৈরি হয়েছিল — আরও বড় machine, আরও বেশি CPU এবং RAM — কারণ একাধিক machine জুড়ে JOIN এবং transaction সঠিকভাবে এবং দ্রুত করা কঠিন। NoSQL database গুলো প্রথম দিন থেকেই মূলত *horizontally* scale করার জন্য তৈরি হয়েছিল — আরও সাধারণ machine যোগ করো, আর database স্বয়ংক্রিয়ভাবে data সেগুলোর মধ্যে ছড়িয়ে দেয় (shard করে)। যদি vertical বনাম horizontal scaling ভিডিওটা মনে থাকে, এটাই ঠিক সেই trade-off যা তোমার database choice-এ সরাসরি দেখা যায়।

**Consistency বনাম availability।** Relational database গুলো ডিফল্টভাবে strong consistency দেয় — তোমার নিজের লেখা তথ্য সাথে সাথে পড়া যায়, সর্বত্র। অনেক distributed NoSQL database ডিফল্টভাবে *eventual* consistency দেয়, "read-your-write" guarantee-র কিছুটা বিনিময়ে network সমস্যার সময় higher availability এবং lower latency পাওয়া যায়। এটি CAP theorem-এর সাথে যুক্ত, যা আমরা এই module-এ পরে বিস্তারিত আলোচনা করব — এখনকার জন্য শুধু জেনে রাখো: SQL consistency-প্রথম দিকে ঝোঁকে, অনেক NoSQL system availability-প্রথম দিকে dial করতে দেয়।

**Query flexibility।** SQL-এর JOIN তোমাকে normalized data জুড়ে যেকোনো, জটিল, ad-hoc প্রশ্ন করতে দেয় — দুর্দান্ত যখন তুমি আগে থেকে তোমার সব access pattern জানো না, যেমন analytics বা reporting-এ। NoSQL সাধারণত তোমাকে *denormalize* করতে বলে — তুমি যেভাবে পড়বে ঠিক সেভাবেই data duplicate করা — যা সাধারণ query গুলোকে অত্যন্ত দ্রুত করে তোলে, কিন্তু ad-hoc, অপরিকল্পিত query এবং cross-record join অনেক কঠিন বা অসম্ভব করে তোলে।

### একটি Decision Framework

তাহলে তুমি আসলে কীভাবে বেছে নেবে? নিজেকে চারটি প্রশ্ন জিজ্ঞেস করো।

**এক — তোমার data দেখতে কেমন?** প্রচুর many-to-many relationship সহ (order, inventory, account) highly structured, relational data স্বাভাবিকভাবেই SQL-এর সাথে মানানসই। Nested, self-contained object (একটি user profile, একটি product catalog entry) document-এর সাথে মানানসই। সহজ lookup (একটি session token, একটি cache entry) key-value-এর সাথে মানানসই। গভীর relationship traversal (social graph, recommendation engine) graph-এর সাথে মানানসই।

**দুই — তোমার scale কেমন?** যদি তুমি স্বাচ্ছন্দ্যে কয়েক মিলিয়ন record এবং একটি ভালোভাবে-tune করা single server-এর মধ্যেই থাকো, SQL তোমাকে বছরের পর বছর ভালো সেবা দেবে — over-engineer কোরো না। যদি write বা storage-এ বিশাল, অনিশ্চিত horizontal growth আশা করো, শুরু থেকেই sharding-এর জন্য designed একটি NoSQL store পরে অনেক কষ্ট বাঁচাবে।

**তিন — strict consistency কতটা গুরুত্বপূর্ণ?** টাকা, inventory count, এমন যেকোনো কিছু যেখানে "stale data পড়া" আসল ক্ষতি করে — SQL এবং strong consistency-র দিকে ঝুঁকো। একটি social media-র like-count বা একটি product view counter — eventual consistency গতি এবং availability-র বিনিময়ে সম্পূর্ণ গ্রহণযোগ্য একটি trade।

**চার — তোমার team ইতিমধ্যে কী জানে, এবং তোমার access pattern কী?** SQL এবং এর query flexibility পরিবর্তনশীল requirement-এর ক্ষেত্রে সহনশীল — একই data-কে নতুন প্রশ্ন করা যায়। NoSQL তোমাকে পুরস্কৃত করে যদি তুমি আগে থেকে তোমার access pattern জানো এবং সেই অনুযায়ী তোমার data-র shape design করো। যদি এখনো নিশ্চিত না হও কীভাবে data query করবে, তাহলে সেই flexibility অনেক মূল্যবান।

### বাস্তব-জগতের উদাহরণ

প্রায় প্রতিটি বড় tech company যে জিনিসটা বুঝে ফেলেছে সেটা হলো: পুরো system-এর জন্য এটা খুব কমই "SQL অথবা NoSQL" — এটা প্রায়ই দুটোই, ইচ্ছাকৃতভাবে প্রয়োগ করা। একে বলা হয় **polyglot persistence**।

Instagram বা Amazon-এর মতো একটি company-র কথা ভাবো। Amazon-এর order এবং payment system — যেখানে একটি transaction অবশ্যই atomic এবং consistent হতে হবে, যেখানে double-charge বা হারিয়ে যাওয়া order গ্রহণযোগ্য নয় — সেগুলো relational database-এ চলে। কিন্তু Amazon-এর shopping cart, বিখ্যাতভাবে, ঐতিহাসিকভাবে DynamoDB-তে চলত, একটি key-value/document store, কারণ cart-গুলোকে বিদ্যুৎ-গতির হতে হয়, network partition-এর সময়ও সবসময় available থাকতে হয়, এবং cross-cart JOIN-এর দরকার হয় না। Instagram মূল social graph — কে কাকে follow করে — দ্রুত traversal-এর জন্য একটি graph-optimized structure-এ store করে, একইসাথে session data এবং feed cache গতির জন্য Redis-এ (key-value) store করে, এবং ঐতিহাসিকভাবে structured account data-র জন্য PostgreSQL ব্যবহার করত। Netflix viewing history এবং telemetry-র বিশাল, বিশ্বব্যাপী distributed write volume সামলাতে Cassandra (wide-column) ব্যবহার করে, আর billing-এর জন্য relational store ব্যবহার করে।

শিক্ষাটা হলো: তোমার পুরো architecture-এর জন্য একজন বিজয়ী বেছে নেওয়ার মতো এটাকে ভেবো না। বরং data-র প্রতিটি *অংশের* জন্য এর shape, scale, এবং consistency চাহিদার উপর ভিত্তি করে সঠিক tool বেছে নেওয়া হিসেবে ভাবো।

### Recap

চলো সব একসাথে করি। SQL database গুলো structured data-কে বাধ্যতামূলক schema এবং relationship সহ table-এ store করে, SQL দিয়ে query করা হয়, এবং ডিফল্টভাবে strong ACID consistency দেয় — যখন তোমার data relational এবং integrity সবচেয়ে গুরুত্বপূর্ণ, তখন আদর্শ। NoSQL চারটি category-র জন্য একটি ছাতা-শব্দ — document, key-value, wide-column, এবং graph — প্রতিটি ভিন্ন data shape-এর জন্য optimize করা এবং horizontally scale করার জন্য তৈরি, প্রায়ই strict consistency-র বিনিময়ে availability এবং গতি পাওয়া। সঠিক পছন্দ নির্ভর করে তোমার data-র shape, তোমার scale, তোমার consistency চাহিদা, এবং তোমার query pattern-এর উপর — এবং প্রকৃত production system গুলো প্রায় সবসময়ই ইচ্ছাকৃতভাবে একাধিক database type মিশিয়ে ব্যবহার করে, শুধু একটি বেছে নেওয়ার বদলে।

### পরবর্তী কী

এখন যেহেতু তুমি জানো *কোন* database model বেছে নিতে হবে, পরের প্রশ্নটা হলো: তোমার table মিলিয়ন বা বিলিয়ন row-এ বেড়ে গেলেও কীভাবে সেই database-এর বিরুদ্ধে query দ্রুত করবে? পরের ভিডিওতে, আমরা ঢুকব **Database Indexing Explained**-এ — B-tree, hash index, এবং কীভাবে সেই "explain query plan" জাদু আসলে ভেতরে ভেতরে কাজ করে। সেখানে দেখা হচ্ছে।

## Key Takeaways

- SQL database গুলো fixed-schema table ব্যবহার করে, foreign key এবং JOIN-এর মাধ্যমে relationship বাধ্যতামূলক করে, এবং strong ACID transaction guarantee প্রদান করে।
- NoSQL-এর চারটি প্রধান category আছে: document (MongoDB), key-value (Redis, DynamoDB), wide-column (Cassandra), এবং graph (Neo4j) — প্রতিটি ভিন্ন data shape-এর জন্য উপযুক্ত।
- SQL strong consistency এবং flexible ad-hoc query-কে প্রাধান্য দেয়; NoSQL horizontal scalability এবং denormalized, access-pattern-optimized data-কে প্রাধান্য দেয়।
- data-র shape, প্রত্যাশিত scale, consistency requirement, এবং জানা query/access pattern-এর ভিত্তিতে বেছে নাও — default অভ্যাস অনুযায়ী নয়।
- বাস্তব-জগতের system গুলো (Amazon, Instagram, Netflix) সাধারণত polyglot persistence ব্যবহার করে: একাধিক database type, প্রতিটি system-এর যে অংশের সাথে সবচেয়ে ভালো মানায় সেই অংশে ব্যবহৃত।
</content>
