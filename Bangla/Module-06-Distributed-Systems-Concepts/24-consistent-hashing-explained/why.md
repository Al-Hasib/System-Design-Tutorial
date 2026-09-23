# Why This Topic Matters: Consistent Hashing

> **এক বাক্যে:** N-টি server জুড়ে key ছড়ানোর সহজ উপায় — `hash(key) % N` — N পরিবর্তিত হওয়ার মুহূর্তেই প্রায় প্রতিটি key remap করে দেয়, যা "একটি cache node যোগ করা"-কে "peak traffic-এ পুরো cache invalidate করা"-য় পরিণত করে।

## The World Before This Idea

আপনার কাছে চারটি cache server আছে। আপনি `hash(key) % 4` দিয়ে key route করেন। এটি সমানভাবে বিতরণ করে এবং এক লাইনের code।

তারপর আপনি একটি পঞ্চম server যোগ করেন। এখন key route হয় `hash(key) % 5` দিয়ে। এর মানে কী তা ভেবে দেখুন: যে key hash হয়ে 7 হয়েছিল সেটি আগে server 3-এ যেত (`7 % 4`), এখন সেটি server 2-এ যায় (`7 % 5`)। সব key-এর জন্য গণনা করলে দেখা যাবে প্রায় **80% key move করে**। এগুলোর প্রতিটিই এখন একটি cache miss।

তাই ঠিক যে মুহূর্তে আপনি load-এর কারণে capacity যোগ করেছিলেন, সেই মুহূর্তেই আপনি আপনার cache-এর 80% invalidate করে সেই traffic database-এর উপর ফেলে দেন। scaling operation-ই আউটেজের কারণ হয়ে দাঁড়ায়। একই ঘটনা ঘটে — আরও খারাপভাবে, অপরিকল্পিতভাবে — যখন একটি server *মারা যায়*।

## The Problems It Solves

### 1. প্রতিটি topology পরিবর্তনে ব্যাপক remapping
**যা আপনি দেখেন:** একটি node যোগ বা অপসারণ একটি cache-miss ঝড় এবং database overload ঘটায়।

**কেন এটি ঘটে:** Modulo hashing প্রতিটি key-এর গন্তব্যকে node-এর *মোট সংখ্যার* সাথে বেঁধে দেয়। সংখ্যা পরিবর্তিত হলে, (প্রায়) প্রতিটি mapping পরিবর্তিত হয়।

**Consistent hashing কীভাবে এটি সমাধান করে:** Key এবং node উভয়কেই একটি hash ring-এ বসানো হয়। একটি key তার clockwise দিকের প্রথম node-এর অন্তর্গত। একটি node যোগ করলে সেটি তার এবং তার predecessor-এর মধ্যবর্তী key-গুলোই ধরে — গড়ে **K/N key**, যেখানে K হলো key-সংখ্যা এবং N হলো node-সংখ্যা। একটি চার-node ring-এ পঞ্চম node যোগ করলে 80%-এর বদলে প্রায় 20% key move করে। একটি node অপসারণ করলে শুধু *তার* key-ই move করে; বাকি সবাই অস্পৃষ্ট থাকে।

### 2. এমন scaling যা করা খুবই বিপজ্জনক
**যা আপনি দেখেন:** টিমগুলো cache বা database node যোগ করা এড়িয়ে চলে কারণ remapping-এর খরচ capacity সমস্যার চেয়েও খারাপ। Capacity planning "চিরকাল peak-এর জন্য provision করা"-য় পরিণত হয়।

**কেন এটি ঘটে:** যদি scaling একটি incident ঘটায়, আপনি scaling বন্ধ করে দেন।

**Consistent hashing কীভাবে এটি সমাধান করে:** এটি elasticity-কে নিরাপদ করে তোলে। Node নিয়মিতভাবে যোগ ও অপসারণ করা যায় — autoscaling, rolling upgrade, বা instance replacement-এর জন্য — কারণ blast radius সীমিত এবং ছোট।

### 3. অসম বিতরণ এবং hotspot
**যা আপনি দেখেন:** ring-এ random-ভাবে বসানো অল্প কয়েকটি node নিয়ে, একটি node একটি বিশাল arc-এর মালিক হয়ে যায় এবং তার প্রাপ্য অংশের চেয়ে অনেক বেশি নেয়। আরও খারাপ, যখন এটি ব্যর্থ হয়, তার *সব* load ঠিক একটি প্রতিবেশীর উপর গিয়ে পড়ে, যা তখন নিজেও ব্যর্থ হয় — একটি cascading failure।

**কেন এটি ঘটে:** একটি circle-এ অল্প সংখ্যক random point অত্যন্ত অসম gap তৈরি করে।

**Virtual node কীভাবে এটি সমাধান করে:** প্রতিটি physical node ring-এ অনেকগুলো point-এ বসানো হয় (100-200 সাধারণ)। বিতরণ নাটকীয়ভাবে মসৃণ হয়ে যায়, এবং একটি node ব্যর্থ হলে তার অনেকগুলো ছোট arc বিভিন্ন প্রতিবেশীর কাছে চলে যায়, load কেন্দ্রীভূত না করে ছড়িয়ে দেয়। Virtual node আপনাকে heterogeneous hardware-কে weight করতেও দেয় — দ্বিগুণ memory-র machine দ্বিগুণ ring position পায়।

### 4. Stateful service যাদের naive-ভাবে load-balance করা যায় না
**যা আপনি দেখেন:** আপনার একজন নির্দিষ্ট user-এর data, session, বা WebSocket connection সব সময় একই node-এ পৌঁছাতে হবে যেখানে সেটি থাকে, কিন্তু round-robin প্রতিটি request ভিন্ন জায়গায় পাঠায়।

**কেন এটি ঘটে:** Standard load balancing ধরে নেয় backend-গুলো পরস্পর-বিনিময়যোগ্য। Stateful backend সেরকম নয়।

**Consistent hashing কীভাবে এটি সমাধান করে:** এটি একটি স্থিতিশীল, deterministic key-to-node mapping প্রদান করে যা প্রতিটি client স্বাধীনভাবে গণনা করতে পারে — কোনো coordinator নেই, কোনো lookup table নেই — এবং যা fleet পরিবর্তিত হলেও স্থিতিশীল থাকে।

## The Price You Pay

- **এক লাইনের modulo-র চেয়ে বেশি জটিলতা।** একটি ring structure, virtual node bookkeeping, এবং সব client-এর বর্তমান membership নিয়ে একমত হওয়ার একটি উপায়।
- **Membership অবশ্যই distributed এবং সম্মত হতে হবে।** প্রতিটি client-এর কাছে ring-এ কোন node আছে তার একটি consistent view থাকতে হবে। যদি দুটি client দ্বিমত পোষণ করে, তারা একই key ভিন্নভাবে route করবে। সেই view কে বর্তমান রাখতে সাধারণত একটি gossip protocol বা একটি coordination service (ZooKeeper, etcd) দরকার হয় — যা নিজেই একটি distributed systems সমস্যা।
- **Hotspot এখনও key-স্তরে থেকে যায়।** Consistent hashing *key distribution* সমান করে; একটি key-কে সেকেন্ডে দশ লক্ষবার request করা নিয়ে এটি কিছুই করে না। একজন celebrity user বা একটি viral item এখনও একটি node-কে overwhelm করে, এবং একটি আলাদা প্রতিকার দরকার (key splitting, hot key-এর local caching, বা dedicated handling)।
- **Rebalancing এখনও data move করে।** K/N বেশিরভাগ ক্ষেত্রেই সবকিছুর চেয়ে অনেক ভালো, কিন্তু একটি বড় dataset-এর জন্য এটি এখনও বাস্তব network traffic এবং বাস্তব সময়, এবং এটি live request পরিবেশন করার সময়ই ঘটে।

## When You Need It — and When You Don't

| যখন এটি ব্যবহার করবেন | যখন এড়িয়ে যাবেন |
|---|---|
| Key node-এ ম্যাপ হয় এবং node set পরিবর্তিত হয় | Node set স্থির এবং কখনো পরিবর্তিত হয় না |
| আপনি একটি cache, shard set, বা partition space বিতরণ করছেন | Backend-গুলো stateless এবং পরস্পর-বিনিময়যোগ্য |
| Node নিয়মিতভাবে ব্যর্থ হয় বা autoscale করে | একটি lookup table বজায় রাখা central coordinator গ্রহণযোগ্য |
| Client-দের একটি central lookup ছাড়াই route করতে হবে | Data volume এত ছোট যে সম্পূর্ণ remapping সস্তা |

## Why This Shows Up in Interviews

Consistent hashing একটি ক্লাসিক interview topic কারণ এটি একটি নির্দিষ্ট, শেখানোর মতো ধারণা যার একটি স্পষ্ট before-and-after আছে, এবং কারণ এটি এমন সিস্টেমের internals-এ দেখা যায় যা সবাই জানে বলে দাবি করে: Cassandra, DynamoDB, Riak, memcached client, Envoy-র ring-hash balancer, এবং বেশিরভাগ CDN request routing। আপনাকে জিজ্ঞাসা করা হতে পারে কেন modulo hashing ব্যর্থ হয়, ring-টি ব্যাখ্যা করতে, এবং virtual node ব্যাখ্যা করতে — শেষেরটিই candidate-রা সবচেয়ে বেশি মিস করে, এবং যেটি বাস্তবে সবচেয়ে গুরুত্বপূর্ণ।

## How It Connects

Consistent hashing হলো সেই mechanism যা **sharding** (topic 14) এবং **distributed caching** (topic 19)-কে operationally টিকিয়ে রাখে, এবং এটিই একটি **load balancer** (topic 7)-কে stateful backend-এ স্থিতিশীলভাবে route করতে দেয়। এটি cluster membership সমস্যা পূর্বধারণা করে যা **consensus algorithm** (topic 27) এবং **coordination service** (topic 40) সমাধান করে। Module 12-এর প্রায় প্রতিটি বড়-মাপের case study-তে আপনি এটি আবার দেখবেন।

**পরবর্তী:** [Rate Limiting Algorithms](../25-rate-limiting-algorithms/why.md) — সেই node-গুলোতে প্রথম স্থানে কতটা load পৌঁছাবে তা নিয়ন্ত্রণ করা।
