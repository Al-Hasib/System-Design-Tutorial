# অনুশীলন ও Interview প্রশ্ন

**১. TCP কোন তিনটি guarantee প্রদান করে যা UDP করে না?**
Reliability (হারানো packet-এর স্বয়ংক্রিয় retransmission সহ নিশ্চিত delivery), ordering (byte/packet গুলো ঠিক যে order-এ পাঠানো হয়েছিল সেই order-এই application-এ deliver হয়), এবং congestion control (network-এ congestion ধরা পড়লে connection ধীর হয়ে যায়)।

**২. একটি live video call সাধারণত TCP-এর বদলে UDP কেন ব্যবহার করে?**
একটি দেরিতে আসা packet (একটি পুরনো, বাসি video frame) একটি হারিয়ে যাওয়া packet-এর চেয়ে খারাপ — একটি হারানো packet retransmit না হওয়া পর্যন্ত TCP তার পরের সবকিছুর delivery থামিয়ে রাখবে, ফলে call থমকে যাবে। UDP শুধু হারানো frame-টা drop করে এবং চালিয়ে যায়, একটি সংক্ষিপ্ত visual glitch-এর বিনিময়ে বিরতিহীন কম latency পায়।

**৩. "Three-way handshake" কী, এবং latency-এর জন্য এটা কেন গুরুত্বপূর্ণ?**
এটা হলো SYN → SYN-ACK → ACK sequence যা কোনো data পাঠানোর আগে connection স্থাপন করতে TCP ব্যবহার করে। এতে application data-এর প্রথম byte সরার আগেই অন্তত একটি সম্পূর্ণ round trip খরচ হয় — এই কারণেই একটি নতুন HTTPS connection আরও ধীর, কারণ এতে একটি TCP handshake এবং তার উপর একটি TLS handshake — দুটোরই খরচ বহন করতে হয়।

**৪. DNS সাধারণত UDP কেন ব্যবহার করে?**
DNS query এবং response সাধারণত ছোট, এবং একটি ছোট request/response-এর জন্য TCP handshake-এর overhead UDP এড়িয়ে যায় — যদি একটি query হারিয়ে যায়, application (resolver) শুধু আবার চেষ্টা করে, যা TCP-এর দরকারি connection setup-এর চেয়ে সামগ্রিকভাবে সস্তা। বড় response-এর জন্য যেখানে reliable, ordered delivery দরকার, সেখানে DNS TCP-তে ফিরে যায়।

**৫. gRPC কী, এবং এটি কোন দুটো জিনিসের উপর তৈরি?**
gRPC হলো Google-এর একটি RPC (Remote Procedure Call) framework যা client-দের remote method-কে যেন local function হয় সেভাবে call করতে দেয়। এটা transport-এর জন্য HTTP/2-এর উপর (multiplexed stream, যা নিজেই TCP-এর উপর চলে) এবং serialization-এর জন্য Protocol Buffers-এর উপর তৈরি।

**৬. Internal service-to-service call-এর জন্য REST/JSON-এর তুলনায় gRPC-এর দুটি সুস্পষ্ট সুবিধা দিন।**
এর যেকোনো দুটো: ছোট, দ্রুত-serialize-হওয়া binary payload (Protocol Buffers বনাম JSON text); WebSockets/SSE জোড়াতালি ছাড়াই streaming-এর (client, server, বা bidirectional) native support; HTTP/2-এর মাধ্যমে multiplexed connection যা per-request connection overhead এড়ায়; client ও server-এর মধ্যে শেয়ার করা একটি strongly typed service contract (.proto file)।

**৭. একটি public, browser-facing API-এর জন্য gRPC সাধারণত কেন খারাপ পছন্দ?**
Browser গুলো plain HTTP/JSON-এর মতো native ভাবে gRPC-এর HTTP/2 framing বলতে পারে না — এর জন্য একটি gRPC-Web proxy layer দরকার — এবং binary Protobuf payload human-readable নয়, যা ad-hoc debugging-কে (যেমন একটি endpoint curl করে response পড়া) REST/JSON-এর চেয়ে অনেক কঠিন করে তোলে। Public API-গুলোও REST-এর ব্যাপক tooling এবং caching support থেকে বেশি উপকৃত হয়।

**৮. Head-of-line blocking কী, এবং এই video-তে কোন protocol এটা এড়ায় এবং কেন?**
Head-of-line blocking হলো যখন data-এর একটি হারানো বা দেরি হওয়া unit তার পেছনে queue-তে থাকা সবকিছুর delivery আটকে দেয়, এমনকি সেই পরের data ইতিমধ্যে পৌঁছে গেলেও। TCP transport layer-এ এই সমস্যায় ভোগে কারণ এটাকে অবশ্যই byte order অনুযায়ী deliver করতে হয়। UDP এটা এড়ায় কারণ প্রতিটি datagram স্বাধীন — একটি হারানো মানে পরেরটার delivery আটকায় না।

**৯. Scenario: আপনি একটি online multiplayer game design করছেন যাতে প্রতি সেকেন্ডে ২০ বার player position update পাঠাতে হয়। আপনি কোন transport বেছে নেবেন, এবং কেন?**
UDP — একটি বাসি position update (হারানো এবং retransmit করা একটি packet থেকে) একটি সামান্য হারিয়ে যাওয়া update-এর চেয়ে খারাপ, যেহেতু পরের update এমনিতেই ৫০ মিলিসেকেন্ডে পৌঁছাবে। TCP-এর retransmission এবং head-of-line-blocking খরচ বহন করার বদলে game তার নিজস্ব হালকা logic (যেমন, সবসময় সবচেয়ে সাম্প্রতিক update বিশ্বাস করা, out-of-order পুরনো update উপেক্ষা করা) বসাতে পারে।

**১০. একজন সহকর্মী আপনার সব internal microservice REST API-কে gRPC দিয়ে replace করার প্রস্তাব দিচ্ছেন। সম্মত হওয়ার আগে আপনি কোন trade-off গুলো তুলবেন?**
Debuggability কমে যায় (binary payload, curl/Postman-এর বদলে gRPC-নির্দিষ্ট tooling দরকার), এটা একটা learning curve এবং `.proto` contract-management overhead যোগ করে, এবং যদি কোনো internal consumer একটি frontend হয় তাহলে এটা সরাসরি browser থেকে কাজ করে না। যদি internal call গুলো high-throughput হয়, streaming দরকার হয়, বা ইতিমধ্যে strict typed contract থেকে উপকৃত হয়, তাহলে এটা একটা শক্তিশালী পছন্দ — সবকিছু migrate করার আগে সেই শর্তগুলো প্রযোজ্য কিনা তা নিশ্চিত করার মতো।

**১১. সত্য বা মিথ্যা: HTTP/3, HTTP/1.1 এবং HTTP/2-এর মতোই সরাসরি TCP-এর উপর চলে।**
মিথ্যা। HTTP/3 QUIC-এর উপর চলে, যা UDP-এর উপর তৈরি — বিশেষভাবে TCP যে head-of-line blocking HTTP/1.1 এবং HTTP/2-এর উপর চাপায় তা এড়াতে, যেখানে QUIC নিজেই reliability এবং ordering user space-এ, প্রতি stream-এ আলাদাভাবে পুনরায় implement করে।
