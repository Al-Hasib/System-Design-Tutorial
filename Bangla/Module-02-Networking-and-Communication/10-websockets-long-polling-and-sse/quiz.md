# Practice & Interview Questions

**১. কেন plain HTTP নিজে থেকে client-কে data push করা সমর্থন করতে পারে না?**
HTTP মূলগতভাবে একটা request-response protocol — server জবাব দেওয়ার আগে client-কেই সবসময় request শুরু করতে হয়। client না চাইতেই server-এর স্বতঃস্ফূর্তভাবে data পাঠানোর কোনো built-in mechanism নেই, আর সে কারণেই workaround টেকনিকগুলো (polling, SSE, WebSockets) তৈরি হয়েছে।

**২. Scale-এ short polling-এর প্রধান অসুবিধা কী?**
এটা প্রতিটা client থেকে নির্দিষ্ট interval-এ একটা constant request-এর স্রোত তৈরি করে, যার বেশিরভাগই কোনো নতুন data ফেরত দেয় না — server এবং network resource-এর অপচয়, আর এটা polling interval দ্বারা সীমাবদ্ধ একটা latency-ও তৈরি করে যেহেতু পরের poll না হওয়া পর্যন্ত নতুন data দেখা যায় না।

**৩. Long polling short polling-এর চেয়ে কীভাবে উন্নত?**
সাথে সাথে "নতুন কিছু নেই" জবাব দেওয়ার বদলে, server request খোলা রাখে যতক্ষণ না প্রকৃতপক্ষে নতুন data পাওয়া যায় (বা একটা timeout ঘটে), তারপর সাথে সাথে জবাব দেয়। এটা অপচয়িত খালি request কমিয়ে দেয় এবং data পাওয়ার মুহূর্তের কাছাকাছি সময়ে সেটা delivery করে।

**৪. Server-Sent Events (SSE) কী, এবং এর প্রধান সীমাবদ্ধতা কী?**
SSE হলো এমন একটা টেকনিক যেখানে client একটা single persistent HTTP connection খোলে এবং server সময়ের সাথে সাথে সেটাতে plain text (`text/event-stream`) আকারে event stream করে। এর প্রধান সীমাবদ্ধতা হলো এটা এক-দিকমুখী — শুধু server থেকে client; client সেই একই stream-এর উপর দিয়ে ফেরত data পাঠাতে পারে না।

**৫. WebSockets-এর প্রেক্ষাপটে "full-duplex" মানে কী, এবং এটা কেন গুরুত্বপূর্ণ?**
Full-duplex মানে হলো client এবং server উভয়ই একই খোলা connection-এর উপর দিয়ে স্বাধীনভাবে এবং একইসাথে একে অপরকে message পাঠাতে পারে। এটা গুরুত্বপূর্ণ কারণ এটা সত্যিকার দুই-দিকমুখী, low-latency interaction (যেমন chat বা multiplayer game) সম্ভব করে, যা request-response-ভিত্তিক টেকনিকগুলো কার্যকরভাবে দিতে পারে না।

**৬. উচ্চ স্তরে WebSocket handshake বর্ণনা করুন।**
Client `Upgrade: websocket` এবং `Connection: Upgrade` header সহ একটা স্বাভাবিক HTTP request পাঠায়। Server যদি এটা সমর্থন করে, তাহলে `101 Switching Protocols` দিয়ে জবাব দেয়, আর সেই মুহূর্ত থেকে TCP connection HTTP semantics থেকে হালকা WebSocket framing protocol-এ পরিবর্তিত হয়, bidirectional message-এর জন্য খোলা থাকে।

**৭. কেন WebSocket connection-এর "sticky" load balancing প্রয়োজন হয়?**
কারণ একটা WebSocket connection stateful এবং দীর্ঘস্থায়ী — একবার স্থাপিত হলে, সেই session-এর সব message একই backend server-এ যেতে হয় যেটা connection ধরে রেখেছে। একটা load balancer-কে connection-এর পুরো জীবনকাল ধরে client-কে সেই একই server-এ route করতে হবে, প্রতিটা message আলাদাভাবে বণ্টন করার বদলে।

**৮. যদি server A-কে এমন একটা client-এর কাছে message পৌঁছাতে হয় যেটা WebSocket-এর মাধ্যমে server B-এর সাথে connected, তাহলে সেটা সাধারণত কীভাবে সমাধান করা হয়?**
একটা pub/sub backplane-এর মাধ্যমে (যেমন, Redis Pub/Sub বা একটা message broker) যা server A-কে message publish করতে দেয়, যেটাতে server B subscribe করে এবং তারপর নিজের স্থানীয়ভাবে connected client-এর কাছে forward করে — কারণ একটা সাধারণ load-balanced setup-এ server A-এর কাছে server B-এর ধরে রাখা connection-এ সরাসরি পৌঁছানোর কোনো উপায় নেই।

**৯. আপনি এমন একটা live sports score notification ফিচার তৈরি করছেন যেখানে user-রা শুধু update পায় এবং সেই channel-এ কখনো ফেরত data পাঠায় না। আপনি কোন টেকনিক বেছে নেবেন, এবং কেন?**
Server-Sent Events — এটা WebSockets-এর চেয়ে implement এবং operate করা সহজ (sticky session বা একটা full-duplex protocol-এর প্রয়োজন নেই), এটা স্বাভাবিক infrastructure-এর মধ্য দিয়ে plain HTTP-এর উপর কাজ করে, আর এর directionality (শুধু server-to-client) প্রয়োজনের সাথে ঠিক মিলে যায়।

**১০. আপনি এমন একটা real-time multiplayer game তৈরি করছেন যেখানে client এবং server উভয়েরই একে অপরকে ঘন ঘন, low-latency update পাঠাতে হবে। কোন টেকনিক সবচেয়ে ভালো মানানসই, এবং কেন?**
WebSockets — game-টার সত্যিকার bidirectional communication দরকার যেখানে ন্যূনতম প্রতি-message overhead এবং low latency দরকার, যা শুধুমাত্র একটা full-duplex, persistent connection দক্ষতার সাথে দিতে পারে; SSE এবং long polling এক-দিকমুখী বা উচ্চতর-latency-এর, এই প্যাটার্নে ভালোভাবে মানানসই হবে না।

**১১. শুধুমাত্র infrastructure/operations দৃষ্টিকোণ থেকে WebSockets-এর তুলনায় SSE-এর একটা সুবিধা কী?**
SSE সরাসরি plain HTTP-এর উপর তৈরি, তাই এটা বিদ্যমান proxy, load balancer, এবং firewall-এর মধ্য দিয়ে বিশেষ কোনো ব্যবস্থা ছাড়াই পার হয়ে যায়, আর browser-গুলো `EventSource`-এর মাধ্যমে স্বয়ংক্রিয় reconnection সামলায় — WebSockets-এর মতো কোনো sticky session প্রয়োজনীয়তা বা custom protocol handling ছাড়াই।

**১২. একটা interview-তে, সবসময় WebSockets-কে default হিসেবে বেছে না নিয়ে long polling-কে একটা fallback টেকনিক হিসেবে আপনি কীভাবে যুক্তিসঙ্গত প্রমাণ করবেন?**
Long polling এমন পরিবেশ বা মধ্যবর্তী infrastructure-এ (পুরোনো proxy, corporate network, নির্দিষ্ট কিছু restrictive firewall) কাজ করে যা হয়তো WebSocket upgrade পুরোপুরি সমর্থন করে না, আর এটা standard request-response HTTP semantics ব্যবহার করে implement করা সহজ — সব client-এর জন্য WebSocket connectivity নিশ্চিত করা না গেলে এটা একটা যুক্তিসঙ্গত degrade-gracefully option।
</content>
