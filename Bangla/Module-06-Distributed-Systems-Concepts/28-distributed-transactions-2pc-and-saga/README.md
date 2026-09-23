# Distributed Transactions: Two-Phase Commit ও Saga Pattern

**কঠিনতার মাত্রা:** Advanced

## শেখার লক্ষ্যসমূহ

- একটি transaction যখন একাধিক service বা database জুড়ে বিস্তৃত হয়, তখন ACID transaction কেন ভেঙে পড়ে তা ব্যাখ্যা করা
- Two-Phase Commit (2PC) প্রোটোকল বিস্তারিতভাবে বর্ণনা করা, যার মধ্যে থাকবে coordinator/participant-এর ভূমিকা, Prepare ও Commit ফেজ, এবং এর blocking failure mode
- Saga pattern ব্যাখ্যা করা, choreography-based এবং orchestration-based saga-র মধ্যে পার্থক্য, এবং compensating transaction কেন rollback নয় বরং semantic undo তা বোঝা
- consistency, availability, complexity, এবং latency-র নিরিখে 2PC ও Saga-র তুলনা করা, এবং বাস্তব architecture-এ কোনটি বেছে নিতে হবে তা জানা
- এই দুই পদ্ধতি বাস্তবায়নে ব্যবহৃত real-world tools ও pattern (XA, Temporal, Camunda, Step Functions) চিনতে পারা

## স্ক্রিপ্ট

### Hook / ভূমিকা

কল্পনা করুন, আপনি একটি ট্রিপ বুক করছেন: একটি ফ্লাইট, একটি হোটেল, এবং একটি রেন্টাল কার। আপনি চান হয় তিনটিই বুক হোক, নয়তো একটিও নয় — আপনি চান না যে প্যারিসে হোটেল রুম আছে কিন্তু সেখানে পৌঁছানোর ফ্লাইট নেই। এটাই distributed transaction সমস্যার মূল সারমর্ম, এবং এটি distributed systems-এর সবচেয়ে জটিল বিষয়গুলোর একটি।

একটি একক database-এ, আপনি এটিকে `BEGIN TRANSACTION ... COMMIT`-এর মধ্যে মুড়ে ACID guarantee-র উপর নির্ভর করতে পারতেন। কিন্তু যখন "ফ্লাইট," "হোটেল," এবং "কার" তিনটি ভিন্ন service, প্রতিটির নিজস্ব database, সম্ভবত তিনটি ভিন্ন টিমের মালিকানাধীন, এমনকি হয়তো তিনটি ভিন্ন কোম্পানির — তখন কী হবে? আপনি শুধু এদের সবার চারপাশে একটি transaction boundary এঁকে আশা করতে পারেন না যে database engine আপনাকে বাঁচাবে। আজ আমরা এই সমস্যার দুটি classic সমাধান নিয়ে আলোচনা করব: Two-Phase Commit, এবং Saga pattern। এই ভিডিওর শেষে আপনি ঠিক জানবেন প্রতিটি কীভাবে কাজ করে, প্রতিটি কোথায় ভেঙে পড়ে, এবং কোনটি আপনার আসলে বেছে নেওয়া উচিত।

### Distributed Transaction কেন কঠিন

চলুন প্রথমে দেখি এটি কেন প্রথম থেকেই কঠিন। ACID — atomicity, consistency, isolation, durability — একটি চমৎকার guarantee, কিন্তু এটি একটি ধারণার উপর প্রতিষ্ঠিত: যে একটি একক transaction manager-এর জড়িত সব data-এর উপর সম্পূর্ণ, synchronous নিয়ন্ত্রণ আছে, সাধারণত একটি database process-এর মধ্যে, কখনো কখনো shard-গুলোর মধ্যে সংযোগকারী একটি দ্রুত local network সহ। এটি row lock করতে পারে, write buffer করতে পারে, এবং হয় সবকিছু atomically disk-এ commit করতে পারে অথবা সবকিছু বাতিল করে দিতে পারে।

একবার আপনার transaction যখন একটি service boundary জুড়ে বিস্তৃত হয় — Order Service, Payment Service, Inventory Service, প্রতিটির নিজস্ব datastore থাকে — তখন আপনি সেই shared context হারিয়ে ফেলেন। সেই local database-গুলো প্রতিটি এখনও আপনাকে local ACID transaction দিতে পারে, কিন্তু কারও কাছেই তিনটির উপর বিস্তৃত একক log বা lock table নেই। এখন আপনাকে নিজেই একটি network-এর মাধ্যমে atomicity coordinate করতে হবে, যা অনির্ভরযোগ্য। একটি request delay হতে পারে, হারিয়ে যেতে পারে, ডুপ্লিকেট হতে পারে, অথবা এটি পরিচালনাকারী মেশিন মাঝপথে crash করতে পারে। এবং সবচেয়ে গুরুত্বপূর্ণ, আপনি "remote service ধীর" এবং "remote service মৃত"-এর মধ্যে পার্থক্য বলতে পারবেন না — এটাই সেই মৌলিক অনিশ্চয়তা যা আমরা এখন আলোচনা করতে যাচ্ছি এমন প্রতিটি distributed transaction প্রোটোকলের কেন্দ্রবিন্দুতে রয়েছে।

তাই আমাদের এমন একটি প্রোটোকল দরকার যা বলে: এখানে দেখুন কীভাবে একাধিক স্বাধীন পক্ষ, একটি অনির্ভরযোগ্য network-এর মাধ্যমে, একটি multi-step operation সম্পূর্ণভাবে ঘটেছে নাকি একেবারেই ঘটেনি তার উপর সম্মত হয় — অথবা, বিকল্পভাবে, এমন একটি strategy যা "সব অথবা কিছুই না" ছেড়ে দিয়ে এর বদলে আরও ক্ষমাশীল কিছু গ্রহণ করে।

### Two-Phase Commit (2PC)

Two-Phase Commit হলো classic, textbook উত্তর, এবং এটি প্রায় একটি local transaction-এর মতোই atomicity বজায় রাখার চেষ্টা করে। এখানে দুটি ভূমিকা রয়েছে: একটি **coordinator** — কখনো কখনো একে transaction manager বলা হয় — এবং এক বা একাধিক **participant**, কখনো কখনো একে resource manager বলা হয়। প্রতিটি participant সাধারণত একটি database বা service যা একটি distributed transaction-এ অংশগ্রহণ করতে সক্ষম।

**Phase 1: Prepare / Vote।** Coordinator প্রতিটি participant-কে একটি "prepare" message পাঠায়, মূলত জিজ্ঞাসা করে, "তুমি কি এটি commit করতে পারবে?" প্রতিটি participant প্রকৃতপক্ষে commit করা ছাড়া বাকি সবকিছু করে ফেলে — এটি প্রয়োজনীয় lock গ্রহণ করে, পরিবর্তনটি একটি durable কিন্তু এখনও দৃশ্যমান নয় এমন log-এ লিখে, constraint যাচাই করে — এবং তারপর একটি vote দিয়ে উত্তর দেয়: "হ্যাঁ, আমি commit করতে পারি," অথবা "না, বাতিল করো।" গুরুত্বপূর্ণ ব্যাপার হলো, একবার একটি participant হ্যাঁ vote দিলে, এটি একটি প্রতিশ্রুতি দিয়ে ফেলেছে: এটিকে পরে অবশ্যই commit করতে সক্ষম হতে হবে, যাই ঘটুক না কেন, এমনকি যদি এটি এই সময়ের মধ্যে crash করে এবং পুনরায় চালু হয়। এই কারণেই vote পাঠানোর আগে prepared state অবশ্যই durably লিখতে হবে।

**Phase 2: Commit / Abort।** Coordinator সব vote সংগ্রহ করে। যদি প্রতিটি participant হ্যাঁ vote দেয়, তাহলে এটি সবাইকে "commit" পাঠায়। যদি এমনকি একটিও না vote দেয় — অথবা সময়মতো সাড়া না দেয় — তাহলে coordinator সবাইকে "abort" পাঠায়, এবং সব participant phase one-এ সাময়িকভাবে staged করা কাজ rollback করে দেয়। এটাই atomicity guarantee: সর্বসম্মত হ্যাঁ অথবা কিছুই ঘটে না।

এখন এখানে বিখ্যাত সমস্যাটি: 2PC একটি **blocking protocol**। ধরুন প্রতিটি participant হ্যাঁ vote দেয়, এবং তারপর coordinator commit message পাঠানোর আগেই crash করে। প্রতিটি participant এখন "prepared" state-এ বসে আছে, lock ধরে রেখেছে, একতরফাভাবে commit বা abort করার সিদ্ধান্ত নিতে অক্ষম — কারণ তাদের যতটুকু জানা, coordinator ইতিমধ্যে অন্য কোনো participant-কে commit করতে বলে থাকতে পারে, এবং যদি তারা নিজে থেকে abort করে, transaction অসঙ্গত হয়ে যাবে। তারা আটকে থাকে, অপেক্ষা করে, live data-র উপর lock ধরে রাখে, যতক্ষণ না coordinator পুনরুদ্ধার হয় বা কোনো মানুষ হস্তক্ষেপ করে। এটি atomicity-র স্বার্থে একটি সরাসরি availability খরচ — এটি Paxos এবং Raft নিয়ে আলোচনার সময় আমরা যে tradeoff-গুলো নিয়ে কথা বলেছিলাম তার একটি বাস্তব-জগতের উদাহরণ: 2PC coordinator failure-এর ক্ষেত্রে একটি consensus protocol-এর মতো fault-tolerant নয়, কারণ এখানে কোনো quorum নেই, কোনো election নেই — এটি একটি একক blocking failure point-সহ একক coordinator। Three-Phase Commit-এর মতো extension আছে যা timeout দিয়ে এটি ঠিক করার চেষ্টা করে, কিন্তু সেগুলো বাস্তবে খুব কমই ব্যবহৃত হয় কারণ সেগুলো blocking-এর বিনিময়ে অন্যান্য জটিলতা তৈরি করে এবং network partition-এর অধীনে এটি সম্পূর্ণভাবে সমাধান করতে পারে না।

আপনি সম্ভবত বাস্তব জগতে 2PC-কে **XA transaction** হিসেবে দেখেছেন — X/Open XA specification হলো একটি standard interface যা একটি transaction manager-কে একাধিক XA-compliant resource, যেমন দুটি ভিন্ন relational database, বা একটি database ও একটি message queue-কে একটি atomic transaction-এর অধীনে coordinate করতে দেয়। PostgreSQL, MySQL, এবং Oracle-এর মতো database XA সমর্থন করে, এবং Java-র JTA (Java Transaction API) এই model-এর উপর ভিত্তি করে তৈরি। এটি কাজ করে, কিন্তু এটি ভারী, এবং এটি আপনার পুরো transaction-এর availability-কে প্রতিটি একক participant এবং coordinator-এর availability ও responsiveness-এর সাথে শক্তভাবে coupled করে দেয়।

### Saga Pattern

Saga pattern সম্পূর্ণ ভিন্ন একটি দর্শন গ্রহণ করে: একটি distributed operation-কে atomic করার চেষ্টা করার বদলে, এটি সেটিকে স্বাধীন **local transaction**-এর একটি sequence-এ ভেঙে ফেলে, যার প্রতিটি নিজের service-এ সাথে সাথেই commit হয়ে যায়। যদি পরবর্তী কোনো step ব্যর্থ হয়, তাহলে database যেভাবে পুরো chain rollback করে সেভাবে না করে, saga ইতিমধ্যে সফল হওয়া প্রতিটি step-এর জন্য **compensating transaction** চালায়, বিপরীত ক্রমে, সেগুলোকে semantically undo করতে।

আমাদের ট্র্যাভেল-বুকিং উপমায় ফিরে আসি: ফ্লাইট বুক করুন — এটি একটি বাস্তব, committed transaction, আপনার সত্যিই একটি টিকেট আছে। হোটেল বুক করুন — এটিও committed। এখন রেন্টাল কার বুকিং ব্যর্থ হয়, ধরুন, কোনো গাড়ি নেই। একটি saga database যেভাবে একটি uncommitted write undo করে সেভাবে ফ্লাইট বুকিং "rollback" করে না — এখানে কোনো undo log নেই কারণ এটি কখনোই uncommitted ছিল না। বরং, এটি একটি compensating action ট্রিগার করে: ফ্লাইট বাতিল করুন, হোটেল বাতিল করুন। এটি একটি business operation — সম্ভবত cancellation fee, refund logic, এবং নিজস্ব failure mode-সহ — কোনো নিম্ন-স্তরের database rollback নয়।

একটি saga coordinate করার দুটি উপায় আছে। **Choreography-based saga**-তে কোনো central coordinator থাকে না: প্রতিটি service তার local transaction করে এবং তারপর একটি event প্রকাশ করে, এবং অন্যান্য service সেই event-এর প্রতিক্রিয়ায় তাদের নিজস্ব local transaction করে এবং তাদের নিজস্ব event প্রকাশ করে। Order Service একটি order তৈরি করে এবং `OrderCreated` emit করে। Payment Service এটি শোনে, কার্ড থেকে টাকা কাটে, এবং `PaymentCompleted` বা `PaymentFailed` emit করে। Inventory Service `PaymentCompleted` শোনে এবং stock reserve করে। যদি Inventory Service `InventoryReservationFailed` emit করে, তাহলে Payment Service এটি শোনে এবং একটি refund জারি করে — এর compensating transaction। এটি loosely coupled এবং আমরা আগে যে event-driven architecture নিয়ে আলোচনা করেছি তার সাথে স্বাভাবিকভাবে কাজ করে, কিন্তু service-এর সংখ্যা বাড়ার সাথে সাথে এটি বোঝা কঠিন হয়ে উঠতে পারে — "transaction" logic প্রতিটি participant-এর event handler জুড়ে ছড়িয়ে থাকে, এবং পুরো flow বোঝার জন্য দেখার মতো একটি একক জায়গা নেই।

**Orchestration-based saga** একটি central orchestrator চালু করে — কোড বা একটি workflow engine-এর একটি অংশ — যা explicitly প্রতিটি service-কে বলে দেয় পরবর্তীতে কী করতে হবে এবং failure-এর সময় explicitly compensating transaction চালায়। Orchestrator Order Service-কে call করে, তারপর Payment Service-কে call করে, তারপর Inventory Service-কে call করে, পুরো পথে state track করতে করতে। যদি Inventory ব্যর্থ হয়, orchestrator explicitly Payment-এর "refund" endpoint এবং Order-এর "cancel" endpoint call করে। এটি logic-কে কেন্দ্রীভূত করে, যা এটিকে test, monitor এবং বোঝা অনেক সহজ করে দেয়, তবে এর বিনিময়ে সেই orchestrator একটি গুরুত্বপূর্ণ infrastructure হয়ে ওঠে — যদিও উল্লেখযোগ্য, যে orchestrator বন্ধ হয়ে গেলেও, ইতিমধ্যে সম্পন্ন হওয়া local transaction-গুলো এখনও বৈধ এবং committed; আপনাকে "শুধু" orchestration state পুনরায় শুরু করতে বা পুনরুদ্ধার করতে হবে, যা 2PC-র blocking সমস্যার তুলনায় মৌলিকভাবে সহজ একটি failure mode, কারণ এই সময় service-গুলো জুড়ে কোনো lock ধরে রাখা হচ্ছে না।

এটা মনে গেঁথে নেওয়া জরুরি যে saga compensation প্রকৃত rollback **নয়**। একটি প্রকৃত rollback ভান করে যে মূল operation কখনো ঘটেইনি, atomically, অন্য কারো কাছে অদৃশ্য। একটি compensating transaction একটি নতুন, আলাদা operation যা মূল operation ইতিমধ্যে জগতের কাছে দৃশ্যমান হওয়ার *পরে* ঘটে — অন্যান্য transaction ইতিমধ্যে সেই intermediate state পড়ে থাকতে পারে। এর মানে হলো saga **isolation** ত্যাগ করে: এমন একটি window থাকে যেখানে ব্যবসায়িক দৃষ্টিকোণ থেকে আংশিক, uncommitted state সিস্টেমের অন্য অংশগুলোর কাছে দৃশ্যমান থাকে। এটি ভালোভাবে সামলানো সত্যিকারের কঠিন design কাজ, যা আমরা takeaway এবং quiz-এ গিয়ে আরও বিস্তারিত আলোচনা করব।

### 2PC বনাম Saga — Tradeoff

চলুন সরাসরি tradeoff-গুলো তুলে ধরি। **Atomicity**: 2PC আপনাকে প্রকৃত atomicity দেয় — সব participant commit করে অথবা কেউই করে না, যা protocol দ্বারা guaranteed। Saga আপনাকে eventual consistency দেয় — sequence শেষ পর্যন্ত হয় "সব committed" অথবা "সব compensated"-এ পৌঁছায়, কিন্তু মাঝখানে একটি window থাকে যেখানে জগৎ আংশিক state-এ থাকে।

**Isolation**: 2PC isolation বজায় রাখে কারণ চূড়ান্ত সিদ্ধান্ত না হওয়া পর্যন্ত পুরো transaction জুড়ে lock ধরে রাখা হয়। Saga মৌলিকভাবে step-গুলো জুড়ে isolation দিতে পারে না — intermediate state দৃশ্যমান থাকে, এই কারণেই anomaly এড়াতে প্রায়ই আপনার semantic lock বা versioning-এর মতো pattern দরকার হয়।

**Availability**: এটাই বড় বিষয়টি। 2PC availability ত্যাগ করে — একটি crashed coordinator lock ধরে রাখা প্রতিটি participant-কে ব্লক করে দেয়। Saga availability বজায় রাখে — প্রতিটি local transaction স্বাধীনভাবে commit হয় এবং এগিয়ে যায়; downstream-এর কোনো failure সবাইকে upstream-এ ব্লক করার বদলে compensation-এর মাধ্যমে asynchronously সামলানো হয়।

**Complexity**: 2PC-র জটিলতা protocol এবং infrastructure-এ থাকে — আপনার XA-compliant resource এবং একটি transaction manager দরকার, কিন্তু application code তুলনামূলকভাবে declarative। Saga জটিলতাকে আপনার application logic-এ ঠেলে দেয় — আপনাকে প্রতিটি step-এর জন্য একটি compensating transaction design ও implement করতে হবে, out-of-order বা duplicate event সামলাতে হবে, এবং compensation-গুলোর নিজেদের আংশিক failure সামলাতে হবে।

**Latency**: 2PC synchronous এবং প্রতিটি participant-এর কাছে একটি network round trip জুড়ে lock ধরে রাখে, যা ধীর এবং বেশি participant বা উচ্চতর latency link-এ ভালোভাবে scale করে না। Saga সাধারণত asynchronous ও non-blocking, তাই সেগুলো ভালো scale করে, যদিও পুরো "transaction" সম্পূর্ণভাবে settle হতে বেশি সময় নিতে পারে।

**Use case**: ছোট সংখ্যক participant-সহ tightly coupled system-এ 2PC বেছে নিন, প্রায়ই একটি একক বিশ্বস্ত infrastructure boundary-র মধ্যে — যেমন একটি monolith দুটি database-এর সাথে কথা বলছে, অথবা স্বল্পস্থায়ী financial ledger operation যেখানে intermediate state দৃশ্যমান হওয়া আপনি সত্যিই সহ্য করতে পারবেন না। Availability এবং loose coupling যেখানে strict atomicity-র চেয়ে বেশি গুরুত্বপূর্ণ, এবং যেখানে business-এর ইতিমধ্যে "cancel" বা "refund"-এর একটি স্বাভাবিক ধারণা আছে, এমন microservice architecture-এ Saga বেছে নিন যা স্বাধীনভাবে deploy করা, স্বাধীনভাবে মালিকানাধীন service জুড়ে বিস্তৃত।

### বাস্তব-জগতের উদাহরণ

সবচেয়ে আদর্শ উদাহরণটি ঠিক সেই একটিই যা দিয়ে আমরা শুরু করেছিলাম, microservices-এ অনুবাদ করা হয়েছে: একটি e-commerce checkout flow যেখানে একটি Order Service, একটি Payment Service, এবং একটি Inventory Service আছে, প্রতিটির নিজস্ব database। এটি বাস্তবে প্রায় সবসময় একটি saga হিসেবে implement করা হয়, 2PC নয়, কারণ এই service-গুলো স্বাধীনভাবে deploy করা হয় এবং আপনি চান না একটি ধীর inventory check payment infrastructure-এর উপর একটি lock ধরে রাখুক। Order একটি "pending" state-এ তৈরি হয়, payment charge করা হয়, inventory reserve করা হয়, এবং শুধুমাত্র তারপরেই order "confirmed" চিহ্নিত করা হয় — যেকোনো step ব্যর্থ হলে cancellation, refund, এবং stock-release compensating action হিসেবে থাকে।

2PC-র দিক থেকে, XA-style distributed transaction এন্টারপ্রাইজ পরিবেশে এখনও খুবই বাস্তব — কিছু relational database এবং message broker (উদাহরণস্বরূপ, নির্দিষ্ট কিছু JMS-compliant broker) XA সমর্থন করে যাতে একটি একক logical transaction একটি database-এ atomically লিখতে পারে এবং একটি message enqueue করতে পারে, নিশ্চিত করে যে আংশিক failure-এর কারণে আপনি কখনো কোনো message হারাবেন না বা দুইবার process করবেন না। আমরা যে availability খরচ নিয়ে আলোচনা করেছি তার কারণে এটি internet স্কেলে অনেক কম common।

Netflix এবং Uber-এর মতো কোম্পানিগুলো তাদের order এবং trip-booking flow-এ saga-র মতো pattern-এর জন্য প্রায়ই উদ্ধৃত হয় — distributed lock-এর পরিবর্তে compensation দিয়ে একাধিক service coordinate করা। এবং বাস্তবে, টিমগুলো ক্রমবর্ধমানভাবে নিজেরা saga orchestration logic হাতে-কলমে তৈরি করে না — তারা **Temporal** বা **Camunda**-র মতো workflow engine-এর দিকে ঝোঁকে, যা আপনাকে durable execution দেয়: আপনি এমন orchestration code লেখেন যা একটি সাধারণ function call sequence-এর মতো দেখায়, এবং engine প্রতিটি step-এর পরে state persist করে যাতে এটি একটি crash-এর পরে ঠিক যেখানে থেমেছিল সেখান থেকেই resume করতে পারে, এবং এতে failure-এর সময় compensating step define ও automatically invoke করার জন্য first-class সমর্থন রয়েছে।

### সারসংক্ষেপ

চলুন এটি একত্রিত করি। একটি distributed transaction সমস্যা তখনই দেখা দেয় যখন atomicity-কে একের বেশি স্বাধীনভাবে ব্যর্থ হতে পারা resource জুড়ে বিস্তৃত হতে হয়। Two-Phase Commit এটি সমাধান করে কিছু commit করার আগে প্রতিটি participant থেকে একটি সর্বসম্মত vote পাওয়ার মাধ্যমে, যা আপনাকে প্রকৃত atomicity এবং isolation দেয়, এর বিনিময়ে coordinator protocol চলাকালীন মারা গেলে প্রতিটি participant ব্লক হয়ে যায় — এই কারণেই এটি ভালোভাবে scale করে না এবং loosely coupled microservices জুড়ে খুব কমই ব্যবহৃত হয়। Saga pattern এর পরিবর্তে প্রতিটি step স্থানীয়ভাবে এবং সাথে সাথেই commit করে এটি সমাধান করে, এবং পরবর্তী কোনো step ব্যর্থ হলে আংশিকভাবে সম্পন্ন কাজ পূর্বাবস্থায় ফেরাতে compensating transaction — একটি semantic undo, প্রকৃত rollback নয় — ব্যবহার করে। Saga-কে coordinate করা যায় choreography দিয়ে, কোনো central authority ছাড়া event ব্যবহার করে, অথবা orchestration দিয়ে, একটি central coordinator ব্যবহার করে যা explicitly প্রতিটি step এবং প্রতিটি compensation চালায়। Saga isolation এবং strict atomicity-র বিনিময়ে availability এবং loose coupling গ্রহণ করে, যা ঠিক সেই বিনিময় যা অধিকাংশ আধুনিক microservice architecture করতে ইচ্ছুক।

### পরবর্তীতে যা আসছে

এখন, saga আপনাকে eventual consistency এবং একটি বাস্তব প্রশ্ন রেখে যায়: যদি intermediate state দৃশ্যমান হয়, এবং যদি একটি network hiccup একটি client-কে একটি request retry করতে বাধ্য করে, তাহলে আপনি কীভাবে নিশ্চিত করবেন যে আপনি একজন customer-কে দুইবার charge করছেন না বা inventory দুইবার reserve করছেন না? আমরা পরবর্তীতে ঠিক এটাই আলোচনা করছি — data consistency model এবং idempotency — তাই থাকুন, কারণ এটি সরাসরি আমরা saga সম্পর্কে যা কিছু আলোচনা করেছি তার উপর ভিত্তি করে গড়ে ওঠে।

## মূল শিক্ষণীয় বিষয়সমূহ

- ACID transaction একটি একক transaction manager-এর উপর নির্ভর করে যার সম্পূর্ণ local নিয়ন্ত্রণ থাকে; একটি transaction যখন অনির্ভরযোগ্য network-এর মাধ্যমে স্বাধীন service বা database জুড়ে বিস্তৃত হয়, তখনই এই ধারণাটি ভেঙে পড়ে।
- Two-Phase Commit একটি coordinator এবং participant ব্যবহার করে: Phase 1 (Prepare/Vote) সবার কাছ থেকে একটি durable প্রতিশ্রুতি পায়, Phase 2 (Commit/Abort) সর্বসম্মত সিদ্ধান্ত কার্যকর করে।
- 2PC একটি blocking protocol — যদি participant-রা হ্যাঁ vote দেওয়ার পরে কিন্তু commit সিদ্ধান্ত পৌঁছানোর আগে coordinator crash করে, তাহলে participant-রা অনির্দিষ্টকালের জন্য lock ধরে আটকে থাকে। এটি একটি বাস্তব availability খরচ, কোনো তাত্ত্বিক বিষয় নয়।
- XA হলো বেশিরভাগ বাস্তব-জগতের 2PC implementation-এর পেছনে থাকা standard specification, যা relational database এবং XA-compliant message broker-এর মতো resource coordinate করে।
- Saga pattern একটি distributed atomic transaction-কে local transaction-এর একটি sequence এবং compensating transaction দিয়ে প্রতিস্থাপন করে যা failure-এর সময় পূর্ববর্তী step-গুলোকে semantically undo করে — compensation নতুন operation, প্রকৃত rollback নয়, এবং সেগুলো isolation পুনরুদ্ধার করে না।
- Choreography-based saga কোনো central authority ছাড়া event-এর মাধ্যমে coordinate করে (loosely coupled, ট্রেস করা কঠিন); orchestration-based saga একটি central orchestrator ব্যবহার করে যা explicitly step এবং compensation চালায় (বোঝা সহজ, কিন্তু orchestrator একটি গুরুত্বপূর্ণ infrastructure)।
- 2PC availability-র বিনিময়ে strong consistency এবং isolation-কে প্রাধান্য দেয়; Saga isolation-এর বিনিময়ে এবং সতর্ক, idempotent compensation logic প্রয়োজন হওয়ার শর্তে availability এবং loose coupling-কে প্রাধান্য দেয়।
- Temporal এবং Camunda-র মতো tool production-এ saga implement করার জন্য বিশেষভাবে তৈরি durable execution engine প্রদান করে, state persistence ও retry logic হাতে-কলমে তৈরি না করে।
