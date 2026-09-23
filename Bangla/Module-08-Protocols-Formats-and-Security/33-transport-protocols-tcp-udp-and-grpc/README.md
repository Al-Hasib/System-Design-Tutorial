# Transport Protocols: TCP vs UDP এবং gRPC কোথায় ফিট করে

**কঠিনতা:** Intermediate

## Learning Objectives

- Networking stack-এ HTTP-এর তুলনায় TCP এবং UDP কোথায় অবস্থান করে তা ব্যাখ্যা করা।
- TCP-এর reliability guarantee গুলো — three-way handshake, ordering, retransmission, এবং congestion control — এবং এগুলোর খরচ কী তা বর্ণনা করা।
- UDP-এর "fire and forget" মডেল এবং reliability ছেড়ে দিয়ে কেন কম latency পাওয়া যায় তা ব্যাখ্যা করা।
- একটি নির্দিষ্ট system design পরিস্থিতির জন্য (web API, video call, DNS, gaming, live streaming) সঠিক transport protocol বেছে নেওয়া।
- gRPC কী, কেন এটি HTTP/2-এর উপর তৈরি, এবং কখন একটি সাধারণ REST API-এর বদলে এটি ব্যবহার করা উচিত তা ব্যাখ্যা করা।

## স্ক্রিপ্ট

### Hook / ভূমিকা

video 6-তে আমরা বলেছিলাম HTTP একটি application-layer protocol যা "TCP/IP-এর উপর বসে থাকে।" আমরা TCP-কে একটি black box হিসেবে ধরে নিয়েছিলাম যা শুধু নির্ভরযোগ্যভাবে এক মেশিন থেকে অন্য মেশিনে byte সরিয়ে নেয়। আজ আমরা সেই box-টি খুলছি। TCP বনাম UDP বোঝা কোনো একাডেমিক তুচ্ছ বিষয় নয় — এটাই কারণ কেন একটি video call-এ একটি frame হারিয়ে গেলেও চলে, কিন্তু একটি bank transfer-এ একটি byte হারালেও চলে না, এবং এটাই কারণ কেন পৃথিবীর কিছু সবচেয়ে high-throughput microservice architecture সম্পূর্ণভাবে REST এড়িয়ে gRPC বেছে নেয়। এই video-এর শেষে, আপনি একটি interview-তে ঠিক ঠিক ব্যাখ্যা করতে পারবেন কেন একটি system TCP, UDP, বা gRPC-এর উপর দাঁড়ানো উচিত — এবং এটা কোনো আন্দাজ হবে না।

### Stack-এ এটি কোথায় বসে

layering-টি মনে করুন: network raw packet (IP) সরায়, একটি transport protocol ঠিক করে সেই packet গুলো দুই endpoint-এর মধ্যে কীভাবে একটি reliable (বা unreliable) data stream-এ পরিণত হবে, এবং HTTP-এর মতো একটি application-layer protocol সেই stream-এর উপরের byte গুলোর অর্থ নির্ধারণ করে। TCP এবং UDP দুটোই transport-layer protocol — "client থেকে server পর্যন্ত byte আসলে কীভাবে পৌঁছায়" এর জন্য এই দুটোই প্রধান দুটি বিকল্প, এবং এই course-এর প্রায় বাকি সবকিছুই এদের একটির উপরে বসে আছে।

### TCP: Reliability, একটি খরচে

TCP — Transmission Control Protocol — connection-oriented। কোনো data সরার আগে, client এবং server একটি three-way handshake সম্পন্ন করে: SYN, SYN-ACK, ACK। সেই handshake একটি stateful connection স্থাপন করে যা দুই পক্ষই track করে, এবং এই কারণেই একটি নতুন TCP connection-এ সবসময় অন্তত একটি অতিরিক্ত round trip খরচ হয়, তার উপরে TLS handshake শুরু করার আগেই (video 6 থেকে HTTPS মনে করুন — সেটা এই TCP handshake-এর উপরে বসানো TLS, তাই একটি একদম নতুন HTTPS connection দুটোরই খরচ বহন করে)।

একবার connect হয়ে গেলে, TCP তিনটি জিনিসের নিশ্চয়তা দেয়: **reliability** — পাঠানো প্রতিটি byte acknowledge করা হয়, এবং হারিয়ে যাওয়া packet স্বয়ংক্রিয়ভাবে retransmit করা হয়; **ordering** — byte গুলো ঠিক যে order-এ পাঠানো হয়েছিল সেই order-এই application-এ পৌঁছায়, এমনকি যদি underlying packet গুলো ভিন্ন path নিয়ে out of order-এ পৌঁছায়; এবং **congestion control** — network-এ congestion ধরা পড়লে TCP সক্রিয়ভাবে গতি কমিয়ে দেয়, যাতে সেই network শেয়ার করা বাকি সবার জন্য পরিস্থিতি আরও খারাপ না হয়। এই কারণেই TCP এমন যেকোনো কিছুর জন্য সঠিক default, যেখানে correctness কয়েক মিলিসেকেন্ড অতিরিক্ত সময়ের চেয়ে বেশি গুরুত্বপূর্ণ: web page, API, file transfer, database connection, email। যদি একটি packet হারিয়ে যায়, TCP নীরবে তা retransmit করে এবং আপনার application এটা জানতেও পারে না যে এমন কিছু ঘটেছিল।

খরচটা হলো latency এবং overhead। Handshake, acknowledgment, এবং retransmission — সবকিছুই সময় নেয়, এবং head-of-line blocking মানে হলো একটি হারানো packet এর পেছনে থাকা সবকিছুকে থামিয়ে দিতে পারে যতক্ষণ না তা recover হয় — HTTP/1.1 এবং HTTP/2-এর জন্য এটি একটি বাস্তব সমস্যা, এবং এটাই একটা কারণ কেন HTTP/3, TCP-এর head-of-line blocking থেকে বাঁচতে বিশেষভাবে QUIC-এ (যা UDP-এর উপর তৈরি) স্থানান্তরিত হয়েছে।

### UDP: Fire and Forget

UDP — User Datagram Protocol — connectionless। এখানে কোনো handshake নেই, নিশ্চিত delivery নেই, নিশ্চিত ordering নেই, এবং স্বয়ংক্রিয় retransmission নেই। আপনি একটি datagram পাঠান, এবং হয় সেটা পৌঁছায় নয়তো পৌঁছায় না — যদি এটা নিয়ে মাথা ঘামাতেই হয়, তাহলে সেটা application-কেই সামলাতে হবে।

এটা শুনতে TCP-এর চেয়ে একেবারেই খারাপ মনে হয়, যতক্ষণ না আপনি একটি video call-এর কথা ভাবছেন। যদি একটি video frame-এর packet হারিয়ে যায়, আপনি কি চান call থমকে যাক যতক্ষণ না TCP সেই একটি packet retransmit করে এবং তার পরের সবকিছু আটকে রাখে? প্রায় কখনোই না — আপনি বরং সেই একটি frame হারাতে চাইবেন এবং live, কিছুটা glitchy video চালিয়ে যেতে চাইবেন, পুরো call-টি থামিয়ে রাখার চেয়ে একটি byte-এর জন্য অপেক্ষা করার বদলে যা পৌঁছানোর সময়ই পুরনো হয়ে যাবে। এটাই মূল বিষয়: যখনই একটি দেরিতে আসা packet একটি হারিয়ে যাওয়া packet-এর চেয়ে খারাপ, তখনই UDP সঠিক পছন্দ। Real-time voice এবং video (VoIP, WebRTC), competitive online gaming (যেখানে আপনি সবচেয়ে সাম্প্রতিক position update চান, দেরিতে আসা পুরনোটা নয়), DNS lookup (যথেষ্ট ছোট যে application level-এ retry করা TCP-এর handshake overhead-এর চেয়ে সস্তা), এবং live streaming protocol — সবাই UDP-এর উপর নির্ভর করে, প্রায়ই শুধু যেখানে সত্যিই প্রয়োজন সেখানে নিজেদের হালকা reliability বা error-concealment বসিয়ে।

### gRPC কোথায় ফিট করে

তাহলে এর মধ্যে gRPC কোথায় ফিট করে? gRPC হলো Google-এর তৈরি একটি আধুনিক RPC (Remote Procedure Call) framework, এবং এটি **HTTP/2**-এর উপর চলে — যা নিজেই TCP-এর উপর চলে। gRPC-এর প্রস্তাব হলো: resource এবং HTTP verb ঘিরে একটি REST API design করার বদলে, আপনি একটি `.proto` file-এ Protocol Buffers ব্যবহার করে একটি **service contract** — typed request/response message সহ একগুচ্ছ remote method — সংজ্ঞায়িত করেন (আমরা পরের video-তে format নিজেই ভালোভাবে কভার করব)। এরপর client সেই method গুলোকে যেন local function-ই হয় সেভাবেই call করে, এবং serialization, network call, এবং deserialization-এর কাজ gRPC নিজেই ভেতরে ভেতরে সামলে নেয়।

সিস্টেমের ভেতরে service-to-service communication-এর জন্য gRPC-কে আকর্ষণীয় করে তোলে তিনটি জিনিস: এটি JSON-এর বদলে Protocol Buffers ব্যবহার করে, যা wire-এ ছোট এবং (de)serialize করতে দ্রুত; এটি HTTP/2-এর multiplexing-এ চলে, তাই অনেক concurrent call একটি connection শেয়ার করে HTTP layer-এ head-of-line blocking ছাড়াই; এবং এতে streaming-এর জন্য first-class support আছে — client-streaming, server-streaming, বা সম্পূর্ণ bidirectional streaming — যা একটি request/response REST call স্বাভাবিকভাবে প্রকাশ করতে পারে না। এর trade-off হলো browser থেকে সরাসরি gRPC call করা কঠিন, debugging-এর জন্য এটি কম human-readable (আপনি এটাকে সরাসরি curl করে response পড়তে পারবেন না), এবং REST/JSON API-এর তুলনায় third-party API consumer-দের কাছে এটি কম পরিচিত। ঠিক এই কারণেই সাধারণ pattern হলো: আপনার নিজের microservice-গুলোর মধ্যে ভেতরে gRPC, public-facing edge-এ REST (বা GraphQL)।

### বাস্তব-জগতের উদাহরণ

একটি ride-sharing backend-এর কথা ভাবুন (যা আমরা Module 9-এ শুরু থেকে শেষ পর্যন্ত design করব)। mobile app HTTPS/REST-এর মাধ্যমে backend-এর সাথে কথা বলে — public-facing, যেকোনো browser বা mobile HTTP client থেকে কাজ করা দরকার, caching এবং পরিচিত tooling থেকে উপকৃত হয়। ভেতরে, location service একটি driver-এর GPS coordinate প্রতি সেকেন্ডে বহুবার matching service-এ stream করে — এটা HTTP/2-এর উপর gRPC-এর server-streaming বা bidirectional streaming-এর জন্য একদম উপযুক্ত, প্রতিটি coordinate update-এর জন্য একটি নতুন REST call খোলার চেয়ে অনেক বেশি efficient। আর location update-গুলোর নিজের নিচে, যদি এটা একটা live-tracking feature হতো যা কাছাকাছি riders-দের app-এ near-real-time position broadcast করত এবং মাঝেমধ্যে একটি update miss হলেও সহনীয় হতো, তাহলে UDP-based transport (বা একটি UDP-backed protocol) হতো সঠিক instinct, যদি latency প্রতিটি একক update পৌঁছানোর চেয়ে বেশি গুরুত্বপূর্ণ হয়।

### সংক্ষিপ্তসার

TCP reliability, ordering, এবং congestion control-এর বিনিময়ে latency ত্যাগ করে — এমন যেকোনো কিছুর জন্য সঠিক default যেখানে correctness raw speed-এর চেয়ে বেশি গুরুত্বপূর্ণ। UDP সেই সব overhead ছেড়ে দিয়ে কম latency পায়, যা তখনই সঠিক trade-off যখন একটি দেরিতে আসা packet একটি হারিয়ে যাওয়া packet-এর চেয়ে খারাপ। gRPC হলো HTTP/2-এর (এবং তাই TCP-এর) উপর তৈরি একটি high-performance RPC framework যা JSON-এর বদলে Protocol Buffers ব্যবহার করে, যা এটাকে internal service-to-service communication-এর জন্য, বিশেষ করে streaming জড়িত থাকলে, একটি শক্তিশালী পছন্দ করে তোলে।

### এরপর কী

আমরা "Protocol Buffers" শব্দগুচ্ছটি বেশ কয়েকবার ব্যবহার করেছি এটাকে সত্যিকারভাবে সংজ্ঞায়িত না করেই। পরের video-তে, আমরা transport থেকে stack-এ এক ধাপ উপরে গিয়ে শুধু message format নিয়ে দেখব — JSON, XML, এবং Protocol Buffers-কে system design-এ যা সত্যিই গুরুত্বপূর্ণ সেই axis গুলোতে তুলনা করব: bandwidth, parsing performance, human-readability, এবং সময়ের সাথে schema পরিবর্তন হলে প্রতিটি কতটা ভালোভাবে সামলায়।

## মূল বিষয়সমূহ (Key Takeaways)

- TCP connection-oriented এবং reliability, ordering, এবং congestion control-এর নিশ্চয়তা দেয় — handshake latency এবং head-of-line blocking-এর খরচে।
- UDP connectionless, কোনো delivery বা ordering guarantee নেই, কম latency-এর বিনিময়ে reliability ত্যাগ করে — এটাই সঠিক পছন্দ যখন একটি দেরিতে আসা packet একটি হারিয়ে যাওয়া packet-এর চেয়ে খারাপ।
- HTTP (এবং HTTPS) TCP-এর উপর চলে; real-time voice/video, gaming, DNS, এবং live streaming সাধারণত UDP-এর দিকে ঝুঁকে থাকে।
- gRPC হলো HTTP/2-এর (TCP-এর উপরে) উপর তৈরি একটি RPC framework যা JSON-এর বদলে Protocol Buffers ব্যবহার করে, যা এটাকে ছোট payload, multiplexed connection, এবং native streaming support দেয়।
- একটি সাধারণ production pattern: compatibility এবং cacheability-এর জন্য public edge-এ REST/JSON, performance এবং streaming-এর জন্য microservice-গুলোর মধ্যে ভেতরে gRPC।
