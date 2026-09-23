# কেন এই বিষয়টি গুরুত্বপূর্ণ: WebSockets, Long Polling & Server-Sent Events

> **এক বাক্যে:** HTTP এমনভাবে ডিজাইন করা হয়েছিল যে client-কেই সবসময় আগে কথা বলতে হয়, যা "কিছু ঘটার সাথে সাথে আমাকে জানাও" এই কাজটাকে আশ্চর্যজনকভাবে কঠিন করে তোলে — আর মানুষ সহজাতভাবে যেসব workaround তৈরি করে, সেগুলোই আসলে সবচেয়ে ব্যয়বহুল।

## এই আইডিয়ার আগের পৃথিবী

আপনি একটা chat app তৈরি করছেন। server-এ একটা message আসে। প্রাপকের browser এটা কীভাবে জানবে?

সহজাত উত্তরটা হলো polling: প্রতি দুই সেকেন্ডে server-কে জিজ্ঞেস করুন, "নতুন কিছু আছে?" এটা কাজ করে, আর এর খরচ নির্মম। ১,০০,০০০ connected user প্রতি ২ সেকেন্ডে poll করলে, তা প্রতি সেকেন্ডে ৫০,০০০ request-এ দাঁড়ায় — আর এর প্রায় ৯৯% "নতুন কিছু নেই" ফেরত দেয়। আপনি একটা খালি উত্তরের জন্য বারবার পুরো HTTP request-এর খরচ (connection handling, TLS, header, auth check, database query) দিচ্ছেন। আর তারপরও user সর্বোচ্চ ২ সেকেন্ডের দেরি দেখছে।

ভালো latency পেতে interval কমিয়ে আনলে load রৈখিকভাবে বাড়ে। Load কমাতে interval বাড়ালে app ভাঙা মনে হয়। এমন কোনো setting নেই যা একইসাথে সস্তা এবং দ্রুত-প্রতিক্রিয়াশীল, কারণ এই সমস্যার জন্য polling কাঠামোগতভাবেই ভুল।

## এটি যেসব সমস্যার সমাধান করে

### ১. খালি poll-এ অপচয়িত কাজ
**আপনি যা দেখবেন:** বিশাল request volume, উচ্চ infrastructure খরচ, আর 200-with-nothing response-এ ভরা dashboard।

**কেন এটা হয়:** client-এর কোনো উপায় নেই জানার যে নতুন খবর আছে কিনা, তাই তাকে ক্রমাগত জিজ্ঞেস করতে হয়।

**push কীভাবে এটা সমাধান করে:** SSE বা WebSockets দিয়ে, connection খোলা থাকে এবং server শুধু তখনই data পাঠায় যখন data থাকে। ১,০০,০০০ idle user-এর খরচ হয় প্রতি সেকেন্ডে ৫০,০০০ request-এর বদলে ১,০০,০০০টা বেশিরভাগ-idle connection। আধুনিক event-loop server idle connection সস্তায় ধরে রাখতে পারে; এটাই পুরো trade।

### ২. Latency floor যা optimize করে সরানো যায় না
**আপনি যা দেখবেন:** backend যত দ্রুতই হোক না কেন, notification-গুলো ধীর মনে হয়।

**কেন এটা হয়:** ২-সেকেন্ডের poll interval-এ, গড় দেরি ১ সেকেন্ড আর worst case ২ সেকেন্ড — server-এর গতির সাথে এর সম্পূর্ণ কোনো সম্পর্ক নেই।

**push কীভাবে এটা সমাধান করে:** event তৈরি হওয়া মাত্রই delivery ঘটে। Latency নেমে আসে network round-trip time-এ, বাস্তবে "real time" বলতে যা বোঝায় সেটাই এটা।

### ৩. Client-to-server stream, শুধু server-to-client নয়
**আপনি যা দেখবেন:** Typing indicator, collaborative cursor, এবং multiplayer game input-এর জন্য *উপরের দিকে* ঘন ঘন ছোট ছোট message দরকার হয়, আর প্রতিটাকে যদি আলাদা HTTP request হিসেবে পাঠানো হয়, তাহলে কয়েক বাইট payload-এর জন্য শত শত বাইট header বহন করতে হয়।

**কেন এটা হয়:** Request/response-এ প্রতি-message নির্দিষ্ট overhead থাকে, যা দুই দিকেই দিতে হয়।

**WebSockets কীভাবে এটা সমাধান করে:** একটাই persistent, full-duplex connection, যার framing overhead single-digit বাইটে পরিমাপযোগ্য। দুই পক্ষই যখন খুশি পাঠাতে পারে। এই ক্ষেত্রেই WebSockets স্পষ্টভাবে সঠিক আর SSE যথেষ্ট নয়।

### ৪. হালকা টুল উপযুক্ত হলেও ভারী টুল বেছে নেওয়া
**আপনি যা দেখবেন:** একটা team একটা live-updating dashboard-এর জন্য WebSockets গ্রহণ করে, তারপর আবিষ্কার করে যে তাদের reconnection, heartbeat, backpressure, একটা server fleet জুড়ে sticky routing, এবং protocol বুঝতে না পারা proxy সামলাতে হবে।

**কেন এটা হয়:** "Real-time"-কে একটাই প্রয়োজন হিসেবে ধরা হয়, যার একটাই উত্তর আছে বলে ধরে নেওয়া হয়।

**এই বিষয়টি কীভাবে এটা সমাধান করে:** এটা কেসগুলোকে আলাদা করে। যদি data শুধু server-to-client প্রবাহিত হয় — notification, live score, progress bar, streaming LLM token — তাহলে Server-Sent Events প্লেইন HTTP, প্রতিটা proxy-র মধ্য দিয়ে কাজ করে, আর প্রায় কোনো কোড ছাড়াই স্বয়ংক্রিয়ভাবে reconnect করে। এটা জানা team-গুলোর মাসের পর মাস আকস্মিক জটিলতা বাঁচিয়ে দেয়।

## যে মূল্য আপনাকে দিতে হবে

Persistent connection একটা সিস্টেম পরিচালনার পদ্ধতি বদলে দেয়:

- **Stateful server।** একটা WebSocket একটা নির্দিষ্ট server-এর সাথে আটকে থাকে। Deploy করলে user disconnect হয়ে যায়; scale out করলে বিদ্যমান connection-গুলো পুনর্বণ্টন হয় না; load balancer-এর connection-aware handling দরকার হয়। এটা statelessness যে সরলতা এনে দিয়েছিল তার কিছুটা মুছে দেয়।
- **Cross-server fan-out।** যদি user A থাকে server 1-এ আর user B থাকে server 3-এ, তাহলে A-এর message B-কে পৌঁছাতে server-গুলোর মধ্যে একটা Pub/Sub backbone (সাধারণত Redis বা Kafka) দরকার হয়।
- **Connection limit এবং memory।** প্রতিটা খোলা connection-এর জন্য file descriptor আর buffer memory খরচ হয়। দশ লক্ষ connection মানে একটা infrastructure project, কোনো config পরিবর্তন নয়।
- **Infrastructure friction।** কিছু corporate proxy আর পুরোনো load balancer দীর্ঘস্থায়ী connection সঠিকভাবে সামলাতে পারে না, তাই production সিস্টেমে fallback path আর মৃত connection শনাক্ত করার জন্য aggressive heartbeat দরকার হয়।
- **Polling মাঝেমধ্যে ঠিকই সঠিক।** কম ঘনঘন update আর কম user সংখ্যার ক্ষেত্রে, একটা সাধারণ poll কম কোড, কম operational ঝুঁকি, এবং একদম পর্যাপ্ত। এমন একটা page-এর জন্য push system তৈরি করবেন না যেটা প্রতি ঘণ্টায় update হয়।

## কখন এটা দরকার — আর কখন নয়

| ব্যবহার | কখন |
|---|---|
| **Short polling** | Update কম ঘনঘন হয়, user সংখ্যা মাঝারি, সরলতাই জেতে |
| **Long polling** | near-real-time দরকার কিন্তু legacy infrastructure-এ কাজ করতে হবে |
| **SSE** | শুধু server-to-client: notification, feed, live metrics, token streaming |
| **WebSockets** | সত্যিকার bidirectional, high-frequency traffic: chat, collaboration, game, trading |

## Interview-এ এটা কেন আসে

Chat, notification, live feed, collaborative editing, আর ride tracking সবই standard design প্রশ্ন, আর প্রতিটাই এই সিদ্ধান্তের উপর নির্ভরশীল। Interviewer যে উত্তরটা চান সেটা "WebSockets" নয় — সেটা হলো reasoning: data কোন দিকে প্রবাহিত হয়, কত ঘনঘন, কতগুলো concurrent connection, আর তাই কোন mechanism। সেরা candidate-রা তারপর নিজে থেকেই কঠিন অংশটা তুলে ধরে: একটা fleet জুড়ে connection state, আর প্রাপকের socket যে server ধরে রেখেছে সেখানে একটা message route করার জন্য প্রয়োজনীয় Pub/Sub layer।

## এটা কীভাবে যুক্ত

Persistent connection সেই statefulness পুনরায় নিয়ে আসে যা দূর করতে **horizontal scaling** কাজ করেছিল, আর সে কারণেই cross-server delivery-র জন্য এগুলো **Pub/Sub**-এর উপর নির্ভরশীল এবং **load balancing** ও **zero-downtime deployment**-কে জটিল করে তোলে। এগুলো **chat application** এবং **ride-sharing** case study-গুলোর কেন্দ্রবিন্দু, আর এই পছন্দটা সরাসরি **transport protocol** এবং **API gateway**-র আচরণের সাথে interact করে।

**পরবর্তী:** [SQL vs NoSQL](../../Module-03-Databases-and-Storage/11-sql-vs-nosql/why.md) — data কীভাবে চলাচল করে সেখান থেকে data কোথায় থাকে সেদিকে সরে যাওয়া।
</content>
