# এই Topic কেন গুরুত্বপূর্ণ: Transport Protocols — TCP vs UDP এবং gRPC কোথায় ফিট করে

> **এক লাইনে:** TCP-এর guarantee গুলো এতটাই সুবিধাজনক যে বেশিরভাগ engineer ভুলে যান এর একটা দাম আছে, আর সেই দাম — head-of-line blocking এবং retransmission delay — ঠিক এই কারণেই একটি video call আটকে যায় বা একটি game ল্যাগ করে বলে মনে হয়।

## এই ধারণার আগের পৃথিবী

একজন application developer যা কিছু ছোঁয় তার প্রায় সবকিছুই TCP-এর উপর চলে, আর TCP সত্যিই চমৎকার: byte গুলো পৌঁছায়, order অনুযায়ী, duplication ছাড়া, নয়তো connection জোরে fail করে। আপনি এটা নিয়ে কখনো ভাবেনও না।

তারপর আপনি কিছু real-time জিনিস তৈরি করেন। TCP-এর উপর একটি voice call: একটি packet হারিয়ে যায়, তাই TCP সেটা retransmit করে এবং তার পরের **প্রতিটি packet-কে আটকে রাখে** যতক্ষণ না হারিয়ে যাওয়াটা এসে পৌঁছায়, কারণ এটাকে অবশ্যই order অনুযায়ী deliver করতে হবে। শ্রোতা নীরবতা শোনে, তারপর একগুচ্ছ পুরনো audio। retransmit করা packet-টিতে ছিল আধা সেকেন্ড আগের ২০ মিলিসেকেন্ড কথা — এটা এখন মূল্যহীন, আর এর জন্য অপেক্ষা করাটা পরের আধা সেকেন্ডকেও নষ্ট করে দিয়েছে।

এটাই head-of-line blocking। TCP-এর reliability guarantee, যা একটি file transfer-এর জন্য ঠিক যা দরকার, তা তখন সরাসরি ক্ষতিকর হয়ে ওঠে যখন data-এর একটি expiry date থাকে।

## এটি যেসব সমস্যা সমাধান করে

### ১. Reliability-এর কারণে নষ্ট হওয়া Real-time data
**আপনি যা দেখেন:** Video call যা থমকে যায় এবং তারপর fast-forward হয়। Multiplayer game যেখানে আপনার position পিছনে চলে যায়। Live stream যা frame drop করার বদলে buffer করে।

**কেন এটা হয়:** TCP packet N+1 deliver করবে না যতক্ষণ না packet N পৌঁছায়, এমনকি যদি N+1 ইতিমধ্যে buffer-এ বসে থাকে এবং N এখন আর প্রাসঙ্গিক না থাকে।

**UDP এটা কীভাবে সমাধান করে:** UDP যা কিছু পৌঁছায় তা সাথে সাথে deliver করে, কোনো ordering বা retransmission ছাড়াই। একটি হারানো audio packet মানে শুধু ২০ মিলিসেকেন্ডের একটি glitch, এবং পরের packet সময়মতো play হয়। যে data-এর মূল্য একটি retransmission round trip-এর চেয়ে দ্রুত কমে যায়, তার জন্য drop করা অপেক্ষা করার চেয়ে সম্পূর্ণভাবে ভালো — এই কারণেই WebRTC, game netcode, VoIP, এবং live streaming — সবাই UDP-এর উপর চলে।

### ২. স্বল্পস্থায়ী request-এ Connection setup latency
**আপনি যা দেখেন:** একটি ছোট API call করতে ৩০০ মিলিসেকেন্ড সময় লাগে, এবং প্রায় পুরোটাই setup — TCP handshake, তারপর TLS handshake, আপনার request-এর একটি byte সরার আগেই।

**কেন এটা হয়:** TCP স্থাপন করতে একটি round trip দরকার; TLS-এর জন্য আরও এক বা দুটো দরকার। একটি cross-region call-এর জন্য, request শুরু হওয়ার আগেই এটা তিনটি round trip।

**এই জ্ঞান এটা কীভাবে সমাধান করে:** এটা ব্যাখ্যা করে কেন connection reuse (keep-alive, connection pooling, HTTP/2 multiplexing) এত গুরুত্বপূর্ণ, এবং কেন QUIC — যা UDP-এর উপর চলে এবং transport ও crypto handshake-কে একটি round trip-এ একত্র করে — তৈরি করার মতো ছিল। খরচটা বোঝা মানেই সমাধানগুলোকে স্বাভাবিক করে তোলে, শুধু অন্ধভাবে অনুসরণ না করে।

### ৩. উচ্চ-volume internal traffic-এ Per-request overhead
**আপনি যা দেখেন:** Internal service গুলো text header parse করা এবং JSON serialize করাতে প্রকৃত CPU খরচ করছে, প্রতি সেকেন্ডে লক্ষ লক্ষ বার।

**কেন এটা হয়:** JSON সহ HTTP/1.1 verbose, এবং একবারে একটি connection-এ একটি request দরকার হয়, তাই client-রা অনেক connection খোলে এবং বারবার setup খরচ বহন করে।

**gRPC এটা কীভাবে সমাধান করে:** এটা HTTP/2-এর উপর চলে, যা একটি connection-এ অনেক concurrent stream multiplex করে — কোনো per-request setup নেই, HTTP layer-এ কোনো head-of-line blocking নেই, এবং compressed binary header। Protocol Buffers-এর সাথে মিলিয়ে, payload আকারে অনেক কম এবং serialization নাটকীয়ভাবে সস্তা। উচ্চ volume-এ service-to-service traffic-এর জন্য, এটা একটা যথেষ্ট, পরিমাপযোগ্য জয়।

### ৪. Streaming RPC pattern যা request/response প্রকাশ করতে পারে না
**আপনি যা দেখেন:** আপনার server থেকে client-এ ক্রমাগত update-এর একটি ধারা push করা দরকার, বা chunk-এর একটি stream upload করা দরকার, আর আপনি এটা polling বা chunked response-এর উপর জোড়াতালি দিয়ে তৈরি করছেন।

**কেন এটা হয়:** Classic HTTP request/response মানে একটি message ভেতরে, একটি message বাইরে।

**gRPC এটা কীভাবে সমাধান করে:** এটা server streaming, client streaming, এবং bidirectional streaming-কে first-class call type হিসেবে প্রস্তাব করে, যা typed client এবং server code-এ generate হয়। এটা সত্যিকারের একটি ভিন্ন capability, শুধু একটা performance optimization নয়।

## যে দাম আপনাকে দিতে হয়

- **UDP reliability-কে আপনার সমস্যা বানিয়ে দেয়।** আপনার যদি কোনো ordering, deduplication, congestion control, বা retransmission দরকার হয়, তাহলে আপনাকেই তা implement করতে হবে — এবং এটা খারাপভাবে করা TCP-এর চেয়েও খারাপ। এই কারণেই আপনার নিজে থেকে লেখার বদলে UDP-এর উপর তৈরি একটি protocol (QUIC, WebRTC, একটি প্রতিষ্ঠিত game networking library) ব্যবহার করা উচিত।
- **UDP প্রায়ই ব্লক করা থাকে।** Corporate firewall এবং কিছু network UDP সীমাবদ্ধ করে, তাই বাস্তব deployment-গুলোতে TCP fallback path দরকার হয়।
- **Edge-এ gRPC ব্যবহার করা অস্বস্তিকর।** Browser গুলো native ভাবে gRPC বলতে পারে না (এর জন্য gRPC-Web প্লাস একটি proxy দরকার), payload human-readable নয়, এবং আপনি `curl` দিয়ে debug করতে পারবেন না বা log-এ একটি request পড়তে পারবেন না। এই কারণেই standard pattern হলো public edge-এ REST/JSON এবং ভেতরে gRPC।
- **Binary contract-এর জন্য tooling discipline দরকার।** Protobuf schema শেয়ার করতে হবে, version করতে হবে, এবং compatible ভাবে evolve করতে হবে। Field number স্থায়ী। একটি schema registry বা shared repository ছাড়া, এটা কষ্টকর হয়ে ওঠে।
- **HTTP/2 head-of-line blocking সরিয়ে দেয় না, শুধু সরিয়ে নেয়।** Stream গুলো HTTP layer-এ multiplex হয়, কিন্তু তারা এখনও একটি TCP connection শেয়ার করে, তাই একটি হারানো TCP segment তাদের সবাইকে থমকে দেয়। QUIC-এর উপর HTTP/3ই আসলে এটা ঠিক করে।

## কখন এটা আপনার দরকার — এবং কখন দরকার নেই

| যখন TCP ব্যবহার করবেন | যখন UDP ব্যবহার করবেন |
|---|---|
| প্রতিটি byte-এর অবশ্যই পৌঁছাতে হবে (file, API, database) | Data দ্রুত মূল্যহীন হয়ে যায় (audio, video, game state) |
| Order গুরুত্বপূর্ণ | Completeness-এর চেয়ে কম latency জেতে |
| আপনি চান OS reliability সামলাক | আপনি ঠিক যতটুকু reliability দরকার ততটুকু নিজে implement করবেন |

| যখন gRPC ব্যবহার করবেন | যখন REST/JSON ব্যবহার করবেন |
|---|---|
| উচ্চ volume-এ internal service-to-service | Public API এবং third-party integration |
| আপনি generated, type-checked contract চান | Browser client এবং সহজ debuggability গুরুত্বপূর্ণ |
| আপনার streaming RPC দরকার | HTTP intermediary-দের caching গুরুত্বপূর্ণ |
| Polyglot service-গুলোর একটি contract definition দরকার | Efficiency-এর চেয়ে simplicity বেশি গুরুত্বপূর্ণ |

## এটা Interview-এ কেন আসে

Video streaming, voice/video calling, gaming, এবং real-time location tracking — এগুলো সবই সাধারণ design prompt, এবং প্রতিটির মূলে থাকে media path-এর জন্য একটি স্পষ্ট justification সহ UDP বেছে নেওয়া। Service side-এ, "আপনার service গুলো কীভাবে একে অপরের সাথে কথা বলে?" প্রশ্নটি gRPC-কে একটি উত্তর হিসেবে ডেকে আনে — কিন্তু শুধু একটি ভালোভাবে যুক্তিসঙ্গত উত্তর: উচ্চ internal volume, typed contract, streaming প্রয়োজন, edge-এ REST রেখে দেওয়ার সাথে সাথে। Head-of-line blocking নির্দিষ্টভাবে ব্যাখ্যা করা এমন একটি পরিষ্কার উপায় যা প্রমাণ করে আপনি আপনার framework-এর নিচের layer বুঝেন।

## এটা কীভাবে সংযুক্ত

এটা **HTTP এবং REST** (topic 6) এবং **WebSockets** (topic 10)-এর নিচের layer — WebSockets TCP-এর উপর চলে, ঠিক এই কারণেই media-এর জন্য WebRTC-এর অস্তিত্ব আছে। **Protocol Buffers** (topic 35) হলো gRPC-এর payload format। **TLS** (topic 36) transport এবং application-এর মাঝখানে বসে। **Web server internals** (topic 34) কভার করে একটি server একই সাথে হাজার হাজার এই ধরনের connection কীভাবে সামলায়, এবং এই পছন্দটা সবচেয়ে বেশি গুরুত্বপূর্ণ হয়ে ওঠে **video streaming** (topic 53) এবং **ride-sharing** (topic 54) case study-তে।

**পরবর্তী:** [Web Server Internals](../34-web-server-internals-concurrency-and-content-serving/why.md) — যখন এই ধরনের দশ হাজার connection একসাথে এসে পৌঁছায় তখন server-এ আসলে কী ঘটে।
