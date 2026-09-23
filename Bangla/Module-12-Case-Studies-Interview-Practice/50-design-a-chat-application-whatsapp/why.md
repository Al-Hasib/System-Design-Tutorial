# এই বিষয়টি কেন গুরুত্বপূর্ণ: একটি Chat Application ডিজাইন করা (WhatsApp-এর মতো)

> **এক বাক্যে:** Chat হলো canonical stateful-connection problem — যে মুহূর্তে আপনি একটি fleet জুড়ে লক্ষ লক্ষ long-lived sockets ধরে রাখেন, stateless horizontal scaling-কে সহজ করে তোলা প্রতিটি ধারণা আর সত্য থাকে না।

## এই Case Study কেন আছে

এই course-এর প্রায় প্রতিটি design problem stateless application servers একটি load balancer-এর পেছনে আছে বলে ধরে নেয়। Chat ইচ্ছাকৃতভাবে সেই ধারণা ভেঙে দেয়।

একজন connected user ঘণ্টার পর ঘণ্টা একটি নির্দিষ্ট server-এ *pinned* থাকে। server 7-এ আসা একটি message-কে server 312-এ ধরে রাখা একটি socket-এ পৌঁছাতে হবে। Deploys মানুষকে disconnect করে দেয়। Autoscaling existing connections-কে rebalance করে না। Load balancer শুধু round-robin করতে পারে না। Servers-কে interchangeable হিসেবে বিবেচনা করা নিয়ে আপনি যা শিখেছেন, তার সবকিছু এমন একটি constraint-এর অধীনে পুনরায় derive করতে হবে যা বলে যে তারা interchangeable নয়।

তার উপরে, chat-এ delivery semantics আছে যা ব্যবহারকারীরা সাথে সাথে লক্ষ্য করেন এবং তীব্রভাবে গুরুত্ব দেন: ordering, sent/delivered/read পার্থক্য, offline delivery, এবং — WhatsApp-এর প্রেক্ষাপটে — end-to-end encryption, যা server-এর কিছু কাজ করার সক্ষমতা কেড়ে নেয় যা আপনি অন্যথায় ধরে নিতেন।

## এটি আপনাকে যে Design Problems সমাধান করতে বাধ্য করে

### ১. recipient-এর socket যে server-এ আছে সেখানে একটি message route করা
**সমস্যা:** User A, server 7-এর সাথে connected। User B, server 312-এ। server 7-এর B-র socket-এ কোনো সরাসরি path নেই।

**কেন এটি কঠিন:** Connection state পুরো fleet জুড়ে distributed এবং ব্যবহারকারীরা connect, disconnect এবং বিভিন্ন servers-এ reconnect করার সাথে সাথে ক্রমাগত পরিবর্তিত হয়।

**আপনি যা শিখবেন:** দুটি standard mechanism, এবং যে বাস্তব systems দুটোই ব্যবহার করে। একটি **session registry** (সাধারণত Redis) user ID-কে সেই server-এ map করে যেটি বর্তমানে তাদের connection ধরে রেখেছে, যাতে server 7 জানতে পারে B কোথায় আছে। একটি **Pub/Sub backbone** server 7-কে একটি channel-এ publish করতে দেয় যা server 312 subscribe করে, topology সম্পর্কে কিছুই জানার প্রয়োজন ছাড়াই। এই layer-টি যে আছে — এবং এটিই আসলে একটি chat architecture-এর মূল কেন্দ্র — এটি বোঝাই এই problem-এর সবচেয়ে গুরুত্বপূর্ণ insight।

### ২. Connection mechanism বেছে নেওয়া, justification সহ
**সমস্যা:** Polling অপচয়ী এবং ধীর; WebSockets শক্তিশালী কিন্তু operationally ভারী।

**আপনি যা শিখবেন:** উত্তরটি traffic shape থেকে আসে। Chat প্রকৃতপক্ষে bidirectional এবং high-frequency (messages, typing indicators, read receipts, presence), তাই এখানে WebSockets সঠিক — একটি notification feed-এর মতো নয়, যেখানে SSE সহজ ও যথেষ্ট হতো। আপনি এটাও শিখবেন WebSockets-এর খরচ কী: connection memory, file descriptor limits, dead connections detect করার জন্য heartbeats, state recovery সহ reconnection, এবং long-lived connections ভুলভাবে handle করে এমন proxies। একটি server কতগুলো connection ধরে রাখতে পারে তা মোটামুটি জানাই আপনার diagram-এ servers-এর সংখ্যা নির্ধারণ করে।

### ৩. যেসব ব্যবহারকারী offline তাদের জন্য messages
**সমস্যা:** B connected নয়। Message হারানো চলবে না, এবং B ফিরে এলে order অনুযায়ী পৌঁছাতে হবে — সম্ভবত একটি ভিন্ন device-এ।

**আপনি যা শিখবেন:** "Real-time delivery" একটি durable store-এর উপর একটি fast path, তার বিকল্প নয়। প্রতিটি message প্রথমে persist করা হয়, তারপর recipient online থাকলে push করা হয়, অথবা একটি mobile push notification-এর মাধ্যমে deliver করা হয় এবং reconnect-এ fetch করা হয়। Data model access pattern থেকে আসে: messages প্রায় সবসময়ই "এই conversation-এর সাম্প্রতিকতম N" হিসেবে read করা হয়, যা conversation ID-কে natural partition key এবং timestamp-কে clustering key বানিয়ে দেয় — একটি wide-column store-এ query-first modeling-এর একটি textbook উদাহরণ।

### ৪. Ordering, deduplication, এবং delivery status
**সমস্যা:** প্রায় একই মুহূর্তে দুটি device থেকে পাঠানো messages, timeout-এর পরে retried, প্রতিটি device-এ একবার, একটি যুক্তিসঙ্গত order-এ দেখাতে হবে।

**কেন এটি কঠিন:** devices এবং servers জুড়ে clocks ভিন্ন, তাই আপনি client timestamp দিয়ে order করতে পারবেন না। Retries মানে duplicates। এবং "delivered" বনাম "read" আলাদা তথ্য যা ফিরে propagate করতে হবে।

**আপনি যা শিখবেন:** **idempotent** delivery-র জন্য client-generated message IDs, wall clocks বিশ্বাস করার বদলে **ordering**-এর জন্য প্রতি conversation-এ server-assigned sequence numbers, এবং tick marks-এর জন্য acknowledgement flows। এখানেই **logical clocks** এবং **idempotency** theory থাকা বন্ধ করে বাস্তব হয়ে ওঠে।

### ৫. Group chat fan-out এবং end-to-end encryption-এর সীমাবদ্ধতা
**সমস্যা:** 500 জনের একটি group-এ একটি message মানে 500টি deliveries। আর messages end-to-end encrypted হলে, server সেগুলো পড়তে পারে না।

**আপনি যা শিখবেন:** Fan-out strategy (প্রতিটি member-এর inbox-এ write করা, নাকি একটি shared conversation log যা members থেকে read করে) এবং এর scaling পরিণতি — news feed problem-এ যে একই fan-out-on-write বনাম fan-out-on-read trade-off প্রাধান্য পায়, ঠিক সেটাই। এরপর E2EE design-কে তীব্রভাবে সীমিত করে দেয়: কোনো server-side search নেই, content-এর উপর কোনো server-side spam filtering নেই, এবং প্রতি recipient device-এর জন্য আলাদা encryption, যা প্রতি message-এ fan-out cost বহুগুণ বাড়িয়ে দেয়। একটি security requirement যে architectural options কেড়ে নেয় তা চিনতে পারা একটি পরিণত observation।

## এটা ভুল করলে যা খরচ হয়

- **Chat servers-কে stateless হিসেবে বিবেচনা করা** এবং একটি message কীভাবে servers পার হয় তা কখনো address না করা এই interview-এর সবচেয়ে বড় ব্যর্থতা।
- **operational cost নিয়ে আলোচনা না করেই WebSockets বেছে নেওয়া** — deploys, reconnection, connection limits — অভিজ্ঞতা ছাড়া শুধু textbook জ্ঞান হিসেবে ধরা পড়ে।
- **Client timestamp দিয়ে order করা** একটি bug যা production-এ সাথে সাথে দেখা দেয় এবং clocks নিয়ে চিন্তা করলে এড়ানো সহজ।
- **Offline users ভুলে যাওয়া** একটি chat app-কে একটি presence-only খেলনায় পরিণত করে।
- **Deduplication উপেক্ষা করা** মানে network-এ সামান্য সমস্যা হলেই ব্যবহারকারীরা প্রতিবার একই message দুইবার দেখেন।

## Interviewers কেন এটি বেছে নেন

Chat একজন candidate-কে আরামদায়ক stateless-service pattern থেকে বের করে connection management, Pub/Sub routing, delivery guarantees, এবং mobile-specific constraints-এর দিকে নিয়ে যায় — তবুও এটি এমন একটি product যা সবাই বোঝে, তাই domain বোঝাতে কোনো সময় নষ্ট হয় না। এটি বেশ কয়েকটি দিকে গভীরে যাওয়ার জন্য যথেষ্ট সমৃদ্ধও, যা এটিকে seniority calibrate করার জন্য একটি ভালো vehicle বানায়: একজন mid-level candidate-এর উত্তর connections এবং storage কভার করে; একজন senior candidate-এর উত্তর fan-out strategy, ordering, idempotency, এবং E2EE কী কেড়ে নেয় তা কভার করে।

## এটি কীভাবে সংযুক্ত

এই case study তৈরি হয়েছে **WebSockets** (topic 10), cross-server routing-এর জন্য **Pub/Sub** (topic 21), session registry এবং presence-এর জন্য **Redis** (topic 19), conversation দিয়ে partitioned **NoSQL** wide-column storage (topics 11, 14), **idempotency and ordering** (topics 29, 41), offline delivery এবং push-এর জন্য **message queues** (topic 20), connection capacity-র জন্য **web server concurrency** (topic 34), connection awareness সহ **load balancing** (topic 7), এবং end-to-end encryption-এর জন্য **security** (topic 36)-এর উপর।

**পরবর্তী:** [Design a News Feed System](../51-design-a-news-feed-system-twitter/why.md) — যেখানে fan-out প্রশ্নটি পুরো architecture হয়ে ওঠে।
