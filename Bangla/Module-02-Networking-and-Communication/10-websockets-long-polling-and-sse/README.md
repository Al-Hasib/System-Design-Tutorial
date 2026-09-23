# WebSockets, Long Polling & Server-Sent Events

**কঠিনতা:** Intermediate

## Learning Objectives

- প্লেইন HTTP request-response কেন real-time ফিচারের ক্ষেত্রে হিমশিম খায়, তা ব্যাখ্যা করা।
- short polling, long polling, Server-Sent Events, এবং WebSockets প্রতিটি কীভাবে কাজ করে তা বর্ণনা করা।
- চারটি টেকনিককে latency, overhead, এবং directionality-এর ভিত্তিতে তুলনা করা।
- WebSocket handshake এবং কেন এটি HTTP থেকে upgrade হয়, তা বোঝা।
- একটি নির্দিষ্ট system design পরিস্থিতির জন্য সঠিক real-time টেকনিক বেছে নেওয়া।

## Script

### Hook / Intro

এই module-এ এতক্ষণ আমরা যা কভার করেছি — HTTP, load balancer, proxy, gateway — সবই একটি নির্দিষ্ট প্যাটার্ন ধরে নেয়: client জিজ্ঞেস করে, server উত্তর দেয়, শেষ। কিন্তু একটা chat app-এর কথা ভাবুন। কেউ আপনাকে একটা মেসেজ পাঠালে, সেটা দেখার জন্য আপনি page refresh করেন না — এটা এমনিতেই চলে আসে। একটা stock ticker, একটা live sports score, বা একটা collaborative document-এ আপনার সহকর্মীর cursor real time-এ নড়াচড়া করার কথা ভাবুন। এগুলোর কোনোটাই "client জিজ্ঞেস করে, server উত্তর দেয়" মডেলে খাপ খায় না, কারণ কিছু ঘটার সাথে সাথেই server-কে client-এর কাছে data push করতে হয়, client না চাইতেই। আজকে আমরা কভার করব এই সমস্যা সমাধানের জন্য production-এ ব্যবহৃত তিনটি আসল টেকনিক: long polling, Server-Sent Events, এবং WebSockets — সেই সাথে এদের কম-উপযোগী পূর্বপুরুষ, short polling — আর ঠিক কখন কোনটা বেছে নেওয়া উচিত।

### The Core Problem: HTTP Wasn't Built for This

আমাদের HTTP video থেকে মনে করুন: এটা একটা request-response protocol। server স্বতঃস্ফূর্তভাবে client-কে data পাঠাতে পারে না — client-কেই সবসময় শুরু করতে হয়। তাহলে এমন একটা protocol থেকে "real-time" আচরণ কীভাবে পাওয়া যায়, যেটার মূল কাঠামোতেই client-কে আগে জিজ্ঞেস করার শর্ত আছে? এর জন্য বেশ কিছু চতুর workaround আছে, প্রতিটির tradeoff ভিন্ন, এবং সহজ সমাধান থেকে জটিল সমাধান পর্যন্ত ক্রমানুসারে এগুলো বোঝাই হলো WebSockets কেন আছে তা সত্যিকারভাবে বোঝার সবচেয়ে ভালো উপায়।

### Short Polling

সহজ সমাধান: client-কে বারবার, চিরকাল জিজ্ঞেস করতে দিন। প্রতি কয়েক সেকেন্ড পরপর, client একটা `GET /messages/new` request পাঠায়। যদি নতুন data থাকে, ভালো, সেটা পেয়ে যায়। না থাকলে, server সাথে সাথে একটা খালি result দিয়ে জবাব দেয়। এরপর client একটা নির্দিষ্ট interval অপেক্ষা করে আবার জিজ্ঞেস করে।

এটা কাজ করে, টেকনিক্যালি, কিন্তু দুইভাবে এটা অপচয়মূলক। প্রথমত, latency: নতুন data যদি একটা poll-এর ঠিক পরেই আসে, তাহলে client পরের poll interval না আসা পর্যন্ত সেটা দেখতে পাবে না — আপনি সবসময় timeliness আর অপচয়কৃত request-এর মধ্যে trade-off করছেন। প্রতি সেকেন্ডে poll করলে বেশি real-time হয় কিন্তু বেশিরভাগ-খালি request দিয়ে server-কে কাহিল করে ফেলে; প্রতি ত্রিশ সেকেন্ডে poll করলে resource বাঁচে কিন্তু response ধীর মনে হয়। দ্বিতীয়ত, এটা গ্রেসফুলভাবে scale করে না: কল্পনা করুন দশ লক্ষ connected client প্রত্যেকে কয়েক সেকেন্ড পরপর poll করছে — এটা আপনার infrastructure-এর উপর একটা constant, বিশাল পরিমাণ বেশিরভাগ-অকেজো request-এর স্রোত। Short polling implement করা সহজ কিন্তু আসল real-time প্রয়োজনের জন্য প্রায় কখনোই সঠিক production উত্তর নয়।

### Long Polling

Long polling এটাকে চতুরভাবে উন্নত করে, একই HTTP request-response mechanism ব্যবহার করেই, কিন্তু server-এর আচরণ পাল্টে দিয়ে। client একটা request পাঠায়, আর server সাথে সাথে "নতুন কিছু নেই" বলে জবাব দেওয়ার বদলে *connection-টা খোলা রাখে* — একদমই জবাব দেয় না — যতক্ষণ না হয় নতুন data পাওয়া যায় অথবা একটা timeout হয়। নতুন কিছু আসামাত্র, server সেই data দিয়ে সাথে সাথে জবাব দেয়। এরপর client, জবাব পাওয়ার সাথে সাথে, তৎক্ষণাৎ একটা নতুন long-poll request খোলে, আর চক্রটা পুনরাবৃত্তি হয়।

Short polling-এর তুলনায় এটা অপচয়কৃত request নাটকীয়ভাবে কমিয়ে দেয় — আপনি ক্রমাগত জিজ্ঞেস করে খালি উত্তর পাচ্ছেন না — এবং নতুন data পাওয়ার মুহূর্তের কাছাকাছি সময়ে সেটা delivery করে, কারণ server-এর কাছে কিছু থাকা মাত্রই সে জবাব দেয়। এর খরচ হলো, hold-এর সময়কাল জুড়ে এটা একটা server connection/thread আটকে রাখে, যা বড় scale-এ server resource-এর উপর চাপ ফেলতে পারে (যদিও আধুনিক async server architecture এটা যুক্তিসঙ্গতভাবে সামলাতে পারে), আর প্রতিটা response-এর পর ক্রমাগত নতুন HTTP connection নতুন করে তৈরি করার কিছু overhead থেকেই যায়।

### Server-Sent Events (SSE)

Server-Sent Events ভিন্ন একটা পদ্ধতি নেয়: বারবার নতুন request খোলার বদলে, client server-এর সাথে একটা single HTTP connection খোলে, আর server সেই connection অনির্দিষ্টকালের জন্য খোলা রাখে, সময়ের সাথে সাথে নতুন event-গুলো client-এর কাছে plain text আকারে একটা নির্দিষ্ট format-এ stream করে, `Content-Type: text/event-stream` ব্যবহার করে। এটা একটা এক-দিকমুখী stream — server চাইলে যখন খুশি সেই একই দীর্ঘস্থায়ী connection-এ client-কে event পাঠাতেই থাকতে পারে, client-কে আবার জিজ্ঞেস করতে হয় না।

SSE-এর সত্যিকার কিছু ভালো বৈশিষ্ট্য আছে: এটা সরাসরি plain HTTP-এর উপর তৈরি, তাই এটা সাধারণ proxy, load balancer, এবং firewall-এর মধ্য দিয়ে বিশেষ কোনো ব্যবস্থা ছাড়াই কাজ করে; browser-এর built-in `EventSource` API connection বিচ্ছিন্ন হলে স্বয়ংক্রিয় reconnection সামলায়; আর server-side implement করা সহজ — আপনি শুধু একটা response stream খোলা রাখছেন এবং সেখানে লিখছেন। এর প্রধান সীমাবদ্ধতা হলো এটা **শুধুমাত্র এক-দিকমুখী** — server থেকে client। client-এর যদি ফেরত data পাঠাতে হয়, তাহলে তাকে SSE stream-এর পাশাপাশি একটা আলাদা, স্বাভাবিক HTTP request ব্যবহার করতে হবে। SSE এমন জিনিসের জন্য দারুণ উপযুক্ত যেমন live notification feed, live score update, অথবা AI-generated text token-বাই-token stream করা, যেখানে server-ই ক্রমাগত update-এর একটা stream তৈরি করছে আর client মূলত শুধু শুনছে।

### WebSockets

WebSockets সমস্যাটার সমাধান একদমই ভিন্নভাবে করে, এবং আরও শক্তিশালীভাবে: এরা client এবং server-এর মধ্যে একটা **persistent, full-duplex** connection স্থাপন করে। Full-duplex মানে হলো দুই পক্ষই একই খোলা connection-এর উপর দিয়ে যে-কোনো সময় স্বাধীনভাবে একে অপরকে message পাঠাতে পারে — এটা আর একদমই request-response নয়।

শুরুটা এভাবে হয়: client একটা স্বাভাবিক HTTP request করে কিন্তু connection "upgrade" করার অনুরোধ জানিয়ে বিশেষ header যোগ করে — `Connection: Upgrade` এবং `Upgrade: websocket`। server যদি এটা সমর্থন করে, তাহলে সে একটা `101 Switching Protocols` status দিয়ে জবাব দেয়, আর তখন underlying TCP connection পুরোপুরি HTTP বলা বন্ধ করে WebSocket protocol-এ পরিবর্তিত হয়ে যায় — এটা একটা হালকা framing protocol যা যে-কোনো পক্ষকে যে-কোনো মুহূর্তে অপর পক্ষকে message পাঠাতে দেয়, একটা full HTTP request-এর তুলনায় প্রতি message-এ খুবই কম overhead সহ।

আপনার যখন সত্যিকার bidirectional, low-latency communication দরকার — chat application, multiplayer game, collaborative editing tool, live trading platform — তখন WebSockets-ই সঠিক টুল। এর tradeoff হলো জটিলতা: কারণ connection-টা stateful এবং দীর্ঘস্থায়ী, এটা এই module-এ আমরা যা কভার করেছি সেই infrastructure-এর সাথে ভিন্নভাবে interact করে। Load balancer-এর "sticky" routing দরকার হয় যাতে একটা client connection-এর পুরো জীবনকাল ধরে একই backend server-এর সাথে সংযুক্ত থাকে, কারণ connection-টা নিজেই state ধরে রাখে। Scale out করার মানে হলো এখন আপনার এমন একটা উপায় দরকার যাতে server A-তে আসা একটা message server B-তে connected client-এর কাছে পৌঁছাতে পারে — যা সাধারণত Redis-এর মতো একটা pub/sub backplane দিয়ে সমাধান করা হয়, যা আমরা Module 5-এ ঠিকভাবে কভার করব। আর WebSocket connection stateless request-response traffic-এর মতো সহজে plain HTTP caching বা সোজাসাপ্টা horizontal autoscaling-এর সাথে খাপ খায় না।

### Comparing All Four

দ্রুত mental model: short polling client-driven এবং অপচয়মূলক। Long polling client-driven কিন্তু efficient, request খোলা রাখে। SSE server-driven, এক-দিকমুখী, এবং সহজ। WebSockets সম্পূর্ণ bidirectional এবং সবচেয়ে শক্তিশালী, কিন্তু scale-এ পরিচালনা করাও সবচেয়ে জটিল। একটা rule of thumb হিসেবে: যদি আপনার শুধু server থেকে client-এ update push করাই দরকার হয়, তাহলে সরলতার জন্য SSE বেছে নিন। যদি সত্যিকার দুই-দিকমুখী, low-latency interaction দরকার হয়, তাহলে WebSockets ব্যবহার করুন। যেসব পরিবেশে WebSockets বা SSE ভালোভাবে সমর্থিত নয়, সেখানে long polling একটা যুক্তিসঙ্গত fallback হিসেবে থেকে যায়।

### Real-World Example

একটা customer support chat widget-এর কথা বিবেচনা করুন। গ্রাহক একটা message টাইপ করে পাঠানো একটা স্বাভাবিক, সহজ action — সেটা আমাদের HTTP video-তে কভার করা সবকিছু ব্যবহার করে শুধু একটা `POST /messages` REST call হতে পারে। কিন্তু support agent যখন জবাব দেয়, তখন গ্রাহকের browser-এ সেই message অবিলম্বে দেখাতে হবে, refresh ছাড়াই। সত্যিকার bidirectional messaging-এর জন্য তৈরি একটা chat product — যেখানে দুই পক্ষই ক্রমাগত ছোট ছোট message আদান-প্রদান করছে — সাধারণত পুরো কথোপকথনের জন্য একটা WebSocket connection ব্যবহার করবে। কিন্তু একটা সহজতর notification ফিচার — যেমন, "একজন agent চ্যাটে যোগ দিয়েছে" বা "আপনার টিকিটের status পরিবর্তিত হয়েছে" — যেখানে server মাঝেমধ্যে এমন একটা client-এর কাছে update push করতে চায় যে সেই channel-এ ফেরত কিছু পাঠাচ্ছে না, সেটা Server-Sent Events-এর জন্য একটা আদর্শ ক্ষেত্র: সহজ infrastructure, sticky session বা একটা পূর্ণ duplex protocol-এর প্রয়োজন নেই।

### Recap

Plain HTTP নিজে থেকে server-কে data push করতে দেয় না — client-কেই সবসময় জিজ্ঞেস করতে হয়। Short polling বারবার জিজ্ঞেস করে এবং অপচয়মূলক। Long polling যতক্ষণ না বলার মতো কিছু থাকে ততক্ষণ request খোলা রাখে, অপচয় কমিয়ে real-time-এর কাছাকাছি থেকে। Server-Sent Events server থেকে client-এ একটা persistent, এক-দিকমুখী stream খোলে, সহজ এবং plain HTTP-এর উপর তৈরি। WebSockets একটা persistent, full-duplex connection খোলে যেখানে যে-কোনো পক্ষ যে-কোনো সময় পাঠাতে পারে — সবচেয়ে শক্তিশালী option, কিন্তু পরিচালনাগতভাবে সবচেয়ে জটিল, যার জন্য scale-এ sticky routing এবং cross-server message delivery দরকার হয়।

### What's Next

এতে Module 2-এর সমাপ্তি ঘটল — আমরা এখন কভার করেছি সিস্টেমগুলো কীভাবে কথা বলে: মৌলিক ভাষা হিসেবে HTTP, traffic এবং shielding layer হিসেবে load balancer ও proxy, microservices orchestrate করা gateway ও BFF, এবং যখন push, pull-এর চেয়ে ভালো তখনকার জন্য real-time protocol। পরবর্তী module-এ, আমরা communication থেকে storage-এ চলে যাব: আমরা শুরু করব SQL vs. NoSQL দিয়ে, আপনার সিস্টেমের আসলে যে data সংরক্ষণ করা দরকার তার জন্য সঠিক database model কীভাবে বেছে নিতে হয় তা গভীরভাবে দেখব।

## Key Takeaways

- Plain HTTP শুধুমাত্র request-response; এই workaround টেকনিকগুলোর একটা ছাড়া server স্বতঃস্ফূর্তভাবে data push করতে পারে না।
- Short polling বারবার server-কে জিজ্ঞেস করে এবং request অপচয় করে; long polling নতুন data না আসা পর্যন্ত request খোলা রাখে, অপচয় কমিয়ে real-time-এর কাছাকাছি থেকে।
- Server-Sent Events plain HTTP-এর উপর একটা সহজ, এক-দিকমুখী (server-to-client) persistent stream দেয়, built-in browser reconnection সাপোর্টসহ।
- WebSockets একটা persistent, full-duplex connection দেয় যেখানে দুই পক্ষই যে-কোনো সময় message পাঠাতে পারে — সবচেয়ে শক্তিশালী, কিন্তু scale করতে sticky load balancing এবং cross-server message delivery দরকার।
- Directionality-এর ভিত্তিতে বেছে নিন: শুধু server-push হলে SSE ভালো; সত্যিকার দুই-দিকমুখী, low-latency interaction হলে WebSockets ভালো; long polling একটা শক্তপোক্ত fallback।
</content>
