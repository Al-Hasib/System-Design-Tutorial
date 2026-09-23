# Study Notes: Transport Protocols — TCP vs UDP এবং gRPC

## সংজ্ঞা

- **Transport layer:** যে networking layer দুই endpoint-এর মধ্যে byte-এর একটি stream (বা sequence) পৌঁছে দেওয়ার দায়িত্বে থাকে; HTTP-এর মতো application protocol-এর নিচে, raw IP packet delivery-এর উপরে বসে।
- **TCP (Transmission Control Protocol):** Connection-oriented transport protocol যা reliable, ordered, congestion-controlled delivery প্রদান করে।
- **UDP (User Datagram Protocol):** Connectionless transport protocol যা best-effort, unordered delivery প্রদান করে, কোনো স্বয়ংক্রিয় retransmission ছাড়াই।
- **gRPC:** Google-এর তৈরি একটি RPC framework, HTTP/2-এর উপর তৈরি, serialization-এর জন্য Protocol Buffers ব্যবহার করে, unary এবং streaming call সাপোর্ট করে।
- **Three-way handshake:** SYN → SYN-ACK → ACK — কোনো data পাঠানোর আগে connection স্থাপন করতে TCP যে setup sequence ব্যবহার করে।
- **Head-of-line blocking:** যখন data-এর একটি হারানো/দেরি হওয়া unit তার পেছনে queue-তে থাকা সবকিছুর delivery আটকে দেয়, এমনকি যদি সেই পরের data ইতিমধ্যে পৌঁছে গিয়ে থাকে।

## TCP vs UDP

| দিক | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (handshake দরকার) | Connectionless |
| Reliability | নিশ্চিত delivery, স্বয়ংক্রিয় retransmission | Best-effort, কোনো retransmission নেই |
| Ordering | নিশ্চিত | নিশ্চিত নয় |
| Congestion control | হ্যাঁ, built-in | নেই (দরকার হলে application-কেই সামলাতে হবে) |
| Overhead | বেশি (handshake, ACK, retransmission) | কম (ন্যূনতম header, কোনো state নেই) |
| Head-of-line blocking | হ্যাঁ, TCP layer-এ | না |
| সাধারণ ব্যবহারের ক্ষেত্র | Web/HTTP, API, database, file transfer, email | VoIP, video call, gaming, DNS, live streaming |

## সাধারণ Protocol গুলো কোথায় বসে

| Protocol | যার উপর চলে | মন্তব্য |
|---|---|---|
| HTTP/1.1, HTTP/2 | TCP | TCP-এর reliability উত্তরাধিকার সূত্রে পায়, তবে এর head-of-line blocking-ও পায় |
| HTTP/3 | QUIC (UDP-এর উপর) | TCP-এর head-of-line blocking এড়াতে user space-এ *প্রতি stream-এ* আলাদাভাবে reliability/ordering পুনর্নির্মাণ করে |
| gRPC | HTTP/2 (→ TCP) | এর উপর Protocol Buffers + native streaming যোগ করে |
| DNS | সাধারণত UDP (বড় response-এর জন্য TCP-তে ফিরে যায়) | ছোট request; app-level retry TCP handshake-এর চেয়ে সস্তা |
| VoIP / WebRTC | UDP | Added latency-এর চেয়ে হারানো packet অনেক বেশি সহনীয় |

## gRPC vs REST (দ্রুত তুলনা)

| দিক | REST (HTTP/1.1 বা 2-এর উপর JSON) | gRPC (HTTP/2-এর উপর Protobuf) |
|---|---|---|
| Payload format | JSON (text) | Protocol Buffers (binary) |
| Payload size / speed | বড়, parse করতে ধীর | ছোট, (de)serialize করতে দ্রুত |
| Browser support | Native (fetch/XHR) | gRPC-Web proxy layer দরকার |
| Human debuggability | বেশি (curl, readable JSON) | কম (binary, tooling দরকার) |
| Streaming | অস্বস্তিকর (WebSockets/SSE দরকার) | Native (client/server/bidirectional streaming) |
| সবচেয়ে উপযুক্ত | Public API, browser client | Internal service-to-service call |

## গুরুত্বপূর্ণ সংখ্যা / তথ্য

- TCP handshake: application data পাঠানোর আগে ১টি round trip (উপরে TLS বসানো থাকলে আরও ২টি, যদিও TLS 1.3 এটা কমাতে পারে)।
- UDP header: ৮ byte, বনাম TCP header: ন্যূনতম ২০ byte — এটাই একটা কারণ কেন UDP-এর প্রতি-packet overhead কম।
- gRPC Google 2015 সালে open-source করেছিল, যা Google-এর ভেতরে "Stubby" নামে ব্যবহৃত একই HTTP/2 stack-এর উপর তৈরি।

## সারসংক্ষেপ

- TCP handshake latency এবং head-of-line blocking-এর খরচে reliability, ordering, এবং congestion control কিনে নেয় — correctness-sensitive traffic-এর জন্য সঠিক default।
- UDP কম latency-এর জন্য সেই সব overhead ছেঁটে ফেলে — সঠিক তখনই যখন একটি দেরিতে আসা packet একটি হারিয়ে যাওয়া packet-এর চেয়ে খারাপ।
- gRPC HTTP/2-এর (TCP-এর) উপর একটি typed RPC contract এবং Protocol Buffers layer করে, যা এটাকে দ্রুত, streaming-সক্ষম internal service communication-এর জন্য একটি শক্তিশালী পছন্দ করে তোলে, যেখানে public, browser-facing API-এর জন্য REST/JSON-ই ভালো পছন্দ থেকে যায়।
