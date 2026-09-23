# এই বিষয়টা কেন গুরুত্বপূর্ণ: Publish-Subscribe Pattern

> **এক বাক্যে:** Point-to-point call মানে হলো কোনো event-এর প্রতিটা নতুন consumer-এর জন্য producer-কে পরিবর্তন করতে হয় — Pub/Sub এটাকে উল্টে দেয়, ফলে একটা listener যোগ করা আপনার কোড পরিবর্তনের বদলে অন্য কারো deployment হয়ে যায়।

## এই ধারণার আগের জগৎ

একজন user একটা order দেয়। order service-কে এখন করতে হবে: card থেকে টাকা কাটা, inventory কমানো, একটা confirmation email পাঠানো, warehouse-কে জানানো, analytics update করা, loyalty point দেওয়া, এবং fraud check করা।

তাহলে `OrderService.placeOrder()` সাতটা service কল করে। ছয় মাস পরে এটা বারোটা কল করে। order service এখন company-র প্রতিটা service সম্পর্কে import করে এবং জানে। প্রতিটা নতুন feature — "একটা SMS-ও পাঠাও," "recommendation engine-কেও জানাও" — order service-এর বিরুদ্ধে একটা pull request হয়ে ওঠে, যা order team review করে, order service-এর release-এ deploy হয়। সেই team এমন feature-এর জন্য একটা bottleneck হয়ে যায় যাতে তাদের কোনো stake নেই, আর ফাইলটা এমন এলোমেলো জিনিসের একটা পড়া-অসম্ভব তালিকা হয়ে যায় যেগুলোর order নিয়ে মাথাব্যথা আছে।

আরও খারাপ, coupling-টা runtime-এও বাস্তব: যদি loyalty service ধীর হয়, তাহলে order placement-ও ধীর হয়ে যায়।

## এটি যে সমস্যাগুলো সমাধান করে

### ১. Producer-কে প্রতিটা consumer সম্পর্কে জানতে হয়
**আপনি যা দেখেন:** একটা service-এর ডজনখানেক outbound dependency আছে, যা প্রতিবার যখন অন্য কোনো team তার কাজের প্রতি সাড়া দিতে চায় তখন পরিবর্তিত হয়।

**কেন এটা ঘটে:** সরাসরি call-এর ক্ষেত্রে caller-কে callee-র নাম বলতে হয়। consumer সম্পর্কে জ্ঞান producer-এর মধ্যেই বেক করা থাকে।

**Pub/Sub কীভাবে এটা সমাধান করে:** producer একটা topic-এ `OrderPlaced` publish করে এবং আর মাথা ঘামায় না। Subscriber-রা নিজেরাই register করে। loyalty team order service-এ হাত না দিয়ে, order team-এর কোনো review ছাড়া, কোনো coordinated deploy ছাড়াই একটা subscriber যোগ করে। organization বড় হওয়ার সাথে সাথে producer-এর কোড বড় হওয়া থেমে যায় — যেটাই আসলে পুরো উদ্দেশ্য।

### ২. Team-জুড়ে coordinated release
**আপনি যা দেখেন:** একটা feature ship করতে তিনটা team-কে একটা নির্দিষ্ট দিনে একটা নির্দিষ্ট order-এ deploy করতে হয়।

**কেন এটা ঘটে:** Compile-time এবং call-time coupling service boundary জুড়ে ছড়িয়ে পড়ে।

**Pub/Sub কীভাবে এটা সমাধান করে:** Publisher এবং subscriber শুধু একটা event schema শেয়ার করে, কোনো code বা call graph নয়। তারা স্বাধীনভাবে deploy করে। এটাই আসলে microservices-কে তার organizational প্রতিশ্রুতি পূরণ করতে দেয় — এটা ছাড়া, আপনি একটা distributed monolith পান যাতে সব জটিলতা আছে কিন্তু কোনো autonomy নেই।

### ৩. একই event একাধিক জায়গায় দরকার, ভিন্ন ভিন্ন semantics-এ
**আপনি যা দেখেন:** Analytics প্রতিটা event চায়, email service প্রতি order-এ একটা email চায়, আর fraud system একটা নতুন model-এর জন্য গত মাসের event replay করতে চায়।

**কেন এটা ঘটে:** একটা point-to-point queue প্রতিটা message ঠিক একজন consumer-কে দেয়। একাধিক স্বাধীন consumer-কে সেবা দেওয়ার মানে হলো নিজে হাতে fan out করা।

**Pub/Sub কীভাবে এটা সমাধান করে:** প্রতিটা subscriber তার নিজের copy এবং stream-এ নিজের position পায়। তারা ভিন্ন ভিন্ন গতিতে consume করে, স্বাধীনভাবে fail করে, এবং (Kafka-র মতো log-based system-এ) যেকোনো offset থেকে replay করতে পারে। একটা publish, অনেক স্বাধীন consumption — যা এই সমস্যার আকৃতির সাথে ঠিক মেলে।

### ৪. Connected client-দের real-time fan-out
**আপনি যা দেখেন:** একটা chat message-কে তিনজন recipient-এর কাছে পৌঁছাতে হবে যাদের WebSocket connection তিনটা আলাদা server ধরে রেখেছে, আর পাঠানো server-এর তাদের কাছে পৌঁছানোর কোনো পথ নেই।

**কেন এটা ঘটে:** Persistent connection নির্দিষ্ট server-এ pin করা থাকে; একটা server-এ আসা message অন্য server-এর socket-এ পৌঁছাতে পারে না।

**Pub/Sub কীভাবে এটা সমাধান করে:** প্রতিটা server তার connected user-দের channel-এ subscribe করে। একটা channel-এ publish করলে সেটা যে server বর্তমানে সেই socket ধরে আছে সেখানে পৌঁছায়। এটাই chat, live notification, এবং collaborative editing-এর জন্য standard মেরুদণ্ড — সাধারণত সরলতার জন্য Redis Pub/Sub অথবা যখন durability এবং replay গুরুত্বপূর্ণ তখন Kafka।

## যে মূল্য আপনাকে দিতে হয়

- **এখন আর কেউ জানে না কী ঘটছে।** producer-এর কোড আর আপনাকে একটা action-এর পরিণতি বলে না। "একটা order দেওয়া হলে কী ঘটে" ট্রেস করতে পুরো সিস্টেমজুড়ে subscription পরীক্ষা করতে হয়। এটাই মূল খরচ, আর এই কারণেই **distributed tracing** ভালো-থাকলে-ভালো হওয়ার বদলে অপরিহার্য হয়ে ওঠে।
- **নীরব ব্যর্থতা (Silent failures)।** যদি কোনো subscriber crash করে বা কখনো deploy-ই না হয়ে থাকে, তাহলে publisher success দেখে। স্পষ্টভাবে কিছু ভুল দেখায় না; জিনিসগুলো কেবল ঘটে না। consumer lag এবং processing failure monitor করা ঐচ্ছিক নয়।
- **Schema evolution এখন একটা distributed সমস্যা।** একটা event-এর আকৃতি পরিবর্তন করলে আপনি এমন consumer ভাঙতে পারেন যাদের অস্তিত্ব সম্পর্কে আপনি জানেনই না। এই কারণেই schema registry এবং শুধু-যোগ-করার (additive-only) field পরিবর্তন গুরুত্বপূর্ণ।
- **Ordering এবং duplicate।** Subscriber-রা partition জুড়ে হয়তো এলোমেলো ক্রমে event দেখে, আর at-least-once delivery মানে duplicate হওয়া। Consumer-দের অবশ্যই idempotent হতে হবে এবং, প্রায়ই, order-tolerant হতে হবে।
- **গঠনগতভাবেই eventual consistency।** publish এবং প্রতিটা consumer catch up করার মধ্যবর্তী সময়ে, সিস্টেমের view-গুলো একমত হয় না। সাধারণত ঠিক আছে; মাঝে মাঝে একটা correctness সমস্যা।

## কখন আপনার এটা দরকার — এবং কখন না

| যখন Pub/Sub ব্যবহার করবেন | যখন সরাসরি call বা point-to-point queue ব্যবহার করবেন |
|---|---|
| একাধিক consumer একই event নিয়ে যত্নশীল | ঠিক একটা সিস্টেমকে কাজ করতে হবে |
| consumer-এর সংখ্যা সময়ের সাথে বাড়বে | আপনার ফলাফল চালিয়ে যাওয়ার দরকার |
| Team-দের স্বাধীনভাবে deploy করতে হয় | operation-টা request-এর সাথে atomic হতে হবে |
| আপনার অনেক client-এর কাছে real-time fan-out দরকার | একটা synchronous answer আবশ্যক (authorization, pricing) |
| আপনি কী ঘটেছে তার replay এবং audit চান | broker যোগ করার খরচ যে coupling সরায় তার চেয়ে বেশি |

## এটা কেন Interview-এ দেখা যায়

Pub/Sub fan-out সমস্যার জন্য প্রত্যাশিত উত্তর: news feed distribution, একটা server fleet জুড়ে chat delivery, notification system, এবং যেখানে একটা action অনেক প্রতিক্রিয়া trigger করে এমন যেকোনো কিছু। Interviewer-রা দেখতে চান আপনি fan-out আকৃতিটা চিনতে পারেন কিনা এবং কঠিন অংশগুলো সামলাতে পারেন কিনা — একটা subscriber down থাকলে কী হবে, double-processing কীভাবে এড়াবেন, ordering guarantee কী, এবং কীভাবে জানবেন একটা consumer পিছিয়ে পড়েছে। বিশেষত chat design-এ, "message কীভাবে recipient-এর connection ধরে রাখা server-এ পৌঁছায়?" এমন একটা প্রশ্ন যার উত্তর Pub/Sub।

## এটা কীভাবে যুক্ত

Pub/Sub হলো সেই delivery model যা **message queue** (topic ২০) বাস্তবায়ন করে এবং যার উপর **event-driven architecture** (topic ২২) নির্মিত। এটা **WebSocket**-এর (topic ১০) জন্য cross-server fan-out layer, সেই communication style যা **microservices**-কে (topic ৩১) decoupled থাকতে দেয়, এবং যে কারণে **idempotency** (topic ২৯) এবং **observability** (topic ৪৩) follow-up-এর বদলে prerequisite। এটা **chat** এবং **news feed** case study-র কেন্দ্রবিন্দু।

**পরবর্তী:** [Event-Driven Architecture](../22-event-driven-architecture/why.md) — যখন আপনি এই ধারণার চারপাশে পুরো সিস্টেম তৈরি করেন তখন কী হয়।
</content>
