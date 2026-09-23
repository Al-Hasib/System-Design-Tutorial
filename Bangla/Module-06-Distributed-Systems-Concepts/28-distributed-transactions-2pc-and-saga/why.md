# এই বিষয়টি কেন গুরুত্বপূর্ণ: Distributed Transactions (2PC ও Saga)

> **এক বাক্যে:** একটি local database transaction আপনাকে একটি database-এর মধ্যে বিনামূল্যে all-or-nothing দেয় — এবং যে মুহূর্তে আপনার operation দুটি database বা দুটি service জুড়ে বিস্তৃত হয়, সেই guarantee উবে যায় এবং আপনাকে সেটি হাতে করে পুনর্নির্মাণ করতে হয়।

## এই ধারণার আগের জগৎ

একটি order-এর জন্য তিনটি জিনিস দরকার: payment service-এ charge করা, inventory কমানো, একটি shipment তৈরি করা। তিনটি service, তিনটি database।

আপনি এগুলোকে ক্রমানুসারে call করেন। Payment সফল হয়। Inventory সফল হয়। Shipment ব্যর্থ হয় — service ডাউন। এখন কী?

Customer-কে এমন একটি order-এর জন্য charge করা হয়েছে যা কখনো shipped হবে না, এবং inventory একটি item কম দেখাচ্ছে যা কারো কাছে নেই। আপনার code-এ কোনো rollback নেই: আপনি একটি HTTP request-কে un-call করতে পারেন না। তাই আপনি payment refund করার জন্য একটি compensating call লেখেন — এবং সেই call-ও ব্যর্থ হয়, কারণ payment service-ও ঠিক তখনই সমস্যায় পড়া শুরু করেছে। এখন টাকা চলে গেছে এবং কেউ জানে না।

প্রতিটি multi-service operation জুড়ে এটিকে গুণ করুন। Reconciliation script, manual support ticket, এবং নীরব অর্থ ক্ষতি এমন system-এর স্বাভাবিক পরিণতি যা কখনো এই সমস্যার সমাধান করেনি।

## এটি যেসব সমস্যার সমাধান করে

### ১. পূর্বাবস্থায় ফেরার কোনো উপায় ছাড়া আংশিক সম্পন্নতা
**আপনি যা দেখেন:** Order অসম্ভব state-এ আছে। পণ্য ছাড়া টাকা সরে গেছে। বাতিল হওয়া order-এর জন্য inventory কমে গেছে।

**এটি কেন ঘটে:** প্রতিটি service স্বাধীনভাবে commit করে। Rollback করার মতো কোনো shared transaction নেই, এবং N নম্বর step-এর পরে একটি failure 1 থেকে N-1 পর্যন্ত step-গুলোকে স্থায়ীভাবে প্রয়োগ করা অবস্থায় রেখে দেয়।

**2PC এটি কীভাবে সমাধান করে:** একটি coordinator একটি **prepare** phase-এ প্রতিটি participant-কে জিজ্ঞাসা করে সে commit করতে পারবে কিনা। প্রতিটি participant কাজটি করে কিন্তু uncommitted রাখে, এবং প্রতিশ্রুতি দেয় সে commit করতে পারবে। শুধুমাত্র যদি সবাই সম্মত হয় তবেই coordinator **commit** পাঠায়। যদি কেউ অস্বীকার করে, সবাই abort করে। এটি system জুড়ে প্রকৃত atomicity দেয়।

**Saga এটি কীভাবে সমাধান করে:** মেনে নেওয়া হয় যে প্রতিটি step সাথে সাথেই commit হয়ে যাবে, এবং প্রতিটির জন্য একটি explicit **compensating action** নির্ধারণ করা হয় — refund charge-কে বিপরীত করে, restock decrement-কে বিপরীত করে। যদি step তিন ব্যর্থ হয়, step দুই এবং এক-এর compensation বিপরীত ক্রমে চালানো হয়। ফলাফল atomic নয় (একটি পর্যবেক্ষণযোগ্য window থাকে যেখানে charge হয়ে গেছে কিন্তু refund হয়নি) কিন্তু এটি অবশেষে সঠিক, এবং এটি কোনো lock ধরে রাখে না।

### ২. Network জুড়ে ধরে রাখা lock
**আপনি যা দেখেন:** Production-এ 2PC থাকলে, একটি coordinator crash participant-দের অনির্দিষ্টকালের জন্য lock ধরে রাখার অবস্থায় ফেলে দেয়। Row জমে যায়, throughput ভেঙে পড়ে, এবং কেউ হস্তক্ষেপ না করা পর্যন্ত system কার্যত বন্ধ থাকে।

**এটি কেন ঘটে:** prepare এবং commit-এর মধ্যে, participant-দের অবশ্যই resource lock ধরে রাখতে হবে — তারা প্রতিশ্রুতি দিয়েছে যে তারা commit করতে পারবে, তাই তারা কাউকে সেই data পরিবর্তন করতে দিতে পারে না। যদি coordinator কখনো সিদ্ধান্ত না পাঠায়, participant-রা আটকে থাকে। এটাই 2PC-র বিখ্যাত **blocking** সমস্যা, এবং তাত্ত্বিকভাবে পরিচ্ছন্ন হওয়া সত্ত্বেও আধুনিক microservice architecture-এ 2PC কেন বিরল তার কারণ এটি।

**Saga এটি কীভাবে সমাধান করে:** কোনো distributed lock একেবারেই নেই। প্রতিটি local transaction সাথে সাথেই commit হয় এবং release হয়। এটাই মূল কারণ যে কেন saga বাস্তবে প্রাধান্য পায়: তারা availability এবং throughput-এর বিনিময়ে atomicity ত্যাগ করে, যা সাধারণত স্কেলে সঠিক বিনিময়।

### ৩. Compensation যা আসলে পূর্বাবস্থায় ফেরাতে পারে না
**আপনি যা দেখেন:** আপনার একটি ইমেইল un-send করতে হবে, একটি রকেট un-launch করতে হবে, অথবা এমন একটি সিট পুনরুদ্ধার করতে হবে যা অন্য কেউ ইতিমধ্যে বুক করে ফেলেছে।

**এটি কেন ঘটে:** প্রতিটি action reversible নয়। Compensation একটি business-স্তরের ধারণা, technical নয়।

**এই বিষয়টি এটি কীভাবে সমাধান করে:** এটি আপনাকে action-এর সাথে সাথে compensation design করতে বাধ্য করে, এবং step-গুলোকে এমনভাবে সাজাতে বাধ্য করে যাতে irreversible step-গুলো সবার শেষে আসে। "সিট reserve করুন, কার্ড থেকে charge করুন, তারপর confirmation ইমেইল পাঠান" একটি saga যা কাজ করে; ইমেইলকে প্রথমে রাখা এমন একটি saga যা কাজ করে না। এই sequencing discipline-ই মূলত ব্যবহারিক দক্ষতার বেশিরভাগ অংশ।

### ৪. Choreography যা কেউ অনুসরণ করতে পারে না
**আপনি যা দেখেন:** একটি saga যা service-গুলো একে অপরের event-এর প্রতিক্রিয়া হিসেবে implement করা হয়েছে, এবং বারোটি step পরে কেউ flow ব্যাখ্যা করতে পারে না, একটি আটকে থাকা instance debug করতে পারে না, বা বলতে পারে না কোন compensation চলেছে।

**এটি কেন ঘটে:** Choreographed saga — যেখানে প্রতিটি service শোনে এবং প্রতিক্রিয়া জানায় — loosely coupled কিন্তু workflow-এর কোনো central record নেই।

**Orchestration এটি কীভাবে সমাধান করে:** একটি নিবেদিত orchestrator state machine ধরে রাখে, প্রতিটি step call করে, এবং compensation চালায়। আপনি কিছুটা decoupling ছেড়ে দেন এবং মূল্যবান কিছু পান: একটি একক জায়গা যা জানে প্রতিটি saga instance কোথায় আছে, যা debug-করা-যায় এমন system এবং ব্যাখ্যাতীত system-এর মধ্যে পার্থক্য। তিন বা চারটি step-এর বেশি যেকোনো কিছুর জন্য, orchestration সাধারণত মূল্যবান।

## আপনি যে মূল্য দিচ্ছেন

- **2PC-র খরচ:** coordinator failure-এ blocking, cross-network lock থেকে দুর্বল throughput, প্রতিটি participant-এর কাছে দুটি round trip-এর latency, এবং একটি coordinator যাকে নিজেই highly available হতে হবে (সাধারণত consensus-এর মাধ্যমে)। এটি এখনও system-এর *ভিতরে* ব্যবহৃত হয় — Kafka transaction, কিছু enterprise stack-এ XA, distributed database-এর অভ্যন্তরে — কিন্তু স্বাধীনভাবে পরিচালিত service জুড়ে এটি একটি খারাপ পছন্দ।
- **Saga-র খরচ:** কোনো isolation একেবারেই নেই। Intermediate state অন্যান্য transaction-এর কাছে দৃশ্যমান, তাই একটি concurrent reader টাকা credit হওয়ার আগেই debit হওয়া দেখতে পারে। এর ফলে সৃষ্ট anomaly প্রতিরোধ করতে semantic lock, pessimistic read, বা compensate করার আগে value পুনরায় পড়ার মতো application-স্তরের কৌশল প্রয়োজন।
- **Compensation code দ্বিগুণ করে।** প্রতিটি step-এর একটি পরীক্ষিত বিপরীত প্রয়োজন। Compensation অবশ্যই idempotent হতে হবে এবং সফল না হওয়া পর্যন্ত নিজেরাই retry করতে হবে — একটি ব্যর্থ compensation system-এর সবচেয়ে খারাপ অবস্থা।
- **Debugging service এবং সময় জুড়ে বিস্তৃত।** এক ঘণ্টা আগে step চার-এ আটকে থাকা একটি saga, দুটি compensation বাকি থাকা অবস্থায়, শক্তিশালী tracing এবং একটি persisted state machine ছাড়া বোঝা সত্যিই কঠিন।

**এবং সবচেয়ে ভালো বিকল্প প্রায়ই কোনোটিই নয়।** যদি দুটি data টুকরো অবশ্যই atomically পরিবর্তিত হতে হয়, সবচেয়ে শক্তিশালী পদক্ষেপ প্রায়ই সেগুলোকে একই service এবং একই database-এ রাখা এবং একটি local transaction ব্যবহার করা। এমনভাবে আঁকা service boundary যেখানে transaction-কে অবশ্যই সেগুলো জুড়ে বিস্তৃত হতে হয়, সাধারণত ভুল জায়গায় আঁকা boundary।

## কখন এটি প্রয়োজন — এবং কখন নয়

| যখন 2PC ব্যবহার করবেন | যখন Saga ব্যবহার করবেন | যখন কোনোটিই ব্যবহার করবেন না |
|---|---|---|
| Participant-রা একটি trust ও ops boundary শেয়ার করে | Step-গুলো স্বাধীনভাবে পরিচালিত service জুড়ে বিস্তৃত | Data একটি database-এই থাকতে পারে |
| Strict atomicity প্রয়োজন | Workflow দীর্ঘস্থায়ী (সেকেন্ড থেকে দিন) | আপনি service boundary পুনরায় আঁকতে পারেন |
| Volume কম এবং latency-সহনশীল | High throughput এবং availability গুরুত্বপূর্ণ | Operation স্বাভাবিকভাবেই একক-service |
| — | Compensating action নির্ধারণযোগ্য | — |

## এটি কেন Interview-এ দেখা যায়

Microservices এবং একটি multi-step business process-সহ যেকোনো design — checkout, booking, ride matching, money transfer — এই সমস্যায় পড়ে। Interviewer-রা জিজ্ঞাসা করেন step তিন ব্যর্থ হলে কী হয়, এবং তারা যাচাই করছেন আপনি জানেন কিনা যে cross-service atomicity বিনামূল্যে পাওয়া যায় না। প্রত্যাশিত উত্তরে saga pattern-এর নাম বলা হয়, compensation নির্দিষ্টভাবে সংজ্ঞায়িত করা হয়, orchestration বনাম choreography উল্লেখ করা হয়, এবং হারানো isolation স্বীকার করা হয়। শক্তিশালী candidate-রা উপরের framing যোগ করেন: প্রথমে জিজ্ঞাসা করেন সেই boundary আদৌ থাকা উচিত কিনা।

## এটি কীভাবে সংযুক্ত

এই সমস্যাটি তৈরি হয় **microservices** (topic 30) এবং **sharding** (topic 14) দ্বারা, দুটোই একক-database transaction ভেঙে দেয়। এটি **ACID vs BASE** (topic 16) এবং **CAP** (topic 15)-এর ব্যবহারিক পরিণতি। Saga **message queue** (topic 20) এবং **event-driven** flow (topic 22)-এর উপর implement করা হয়, এবং সেগুলোর অবশ্যই **idempotency** (topic 29) প্রয়োজন, কারণ প্রতিটি step এবং compensation retry হবে। 2PC coordinator-কে highly available হতে সাধারণত **consensus** (topic 27)-এর উপর নির্ভর করতে হয়, এবং **domain-driven design** (topic 32) হলো এমন boundary আঁকার discipline যা এই সবকিছুর প্রয়োজনীয়তা কমিয়ে দেয়।

**পরবর্তী:** [Data Consistency Models & Idempotency](../29-data-consistency-models-and-idempotency/why.md) — এই সবকিছুর নিচে থাকা guarantee এবং safety net।
