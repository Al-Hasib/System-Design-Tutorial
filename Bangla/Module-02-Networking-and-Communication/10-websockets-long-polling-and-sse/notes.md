# Study Notes: WebSockets, Long Polling & Server-Sent Events

## Definitions

- **Short polling:** Client নির্দিষ্ট interval-এ বারবার request পাঠায় নতুন data আছে কিনা চেক করতে; server সাথে সাথে জবাব দেয় (প্রায়ই খালি)।
- **Long polling:** Client একটা request পাঠায়; server সেটা খোলা রাখে যতক্ষণ না নতুন data পাওয়া যায় বা একটা timeout হয়, তারপর জবাব দেয়; client তৎক্ষণাৎ একটা নতুন request পুনরায় খোলে।
- **Server-Sent Events (SSE):** একটা single, দীর্ঘস্থায়ী HTTP connection যার মাধ্যমে server client-কে এক-দিকমুখীভাবে event stream করে, `Content-Type: text/event-stream` ব্যবহার করে।
- **WebSocket:** একটা persistent, full-duplex TCP-ভিত্তিক connection, যা একটা HTTP "Upgrade" handshake-এর মাধ্যমে শুরু হয়, যা যে-কোনো পক্ষকে যে-কোনো সময় message পাঠাতে দেয়।
- **Full-duplex:** দুই পক্ষই একই connection-এর উপর দিয়ে একইসাথে এবং স্বাধীনভাবে পাঠাতে ও গ্রহণ করতে পারে।

## Comparison Table

| টেকনিক | Directionality | Latency | Server overhead | জটিলতা | কীসের উপর তৈরি |
|-----------|------------------|---------|--------------------|-------------|-----------|
| Short polling | Client-driven (pull) | খারাপ (poll interval দ্বারা সীমাবদ্ধ) | বেশি (অনেক বেশিরভাগ-খালি request) | কম | Plain HTTP request |
| Long polling | Client-driven (pull), server response দেরি করে | ভালো (near real-time) | মাঝারি (connection খোলা রাখে) | মাঝারি | Plain HTTP, খোলা-রাখা request |
| Server-Sent Events (SSE) | শুধু Server → Client | ভালো (server সাথে সাথে push করে) | কম-মাঝারি (প্রতি client-এ একটা persistent stream) | কম-মাঝারি | Plain HTTP streaming (`text/event-stream`) |
| WebSocket | Full-duplex (উভয় দিকে) | চমৎকার (প্রতি-message ন্যূনতম overhead) | মাঝারি-বেশি (persistent stateful connection) | বেশি | HTTP Upgrade → নিবেদিত WebSocket protocol |

## WebSocket Handshake

1. Client `Connection: Upgrade` এবং `Upgrade: websocket` header (সাথে `Sec-WebSocket-Key`) সহ একটা HTTP request পাঠায়।
2. Server `101 Switching Protocols` (সাথে `Sec-WebSocket-Accept`) দিয়ে জবাব দেয়।
3. Connection HTTP semantics থেকে WebSocket framing protocol-এ পরিবর্তিত হয়।
4. এখন থেকে যে-কোনো পক্ষ যে-কোনো সময় framed message পাঠাতে পারে, যতক্ষণ না connection বন্ধ হয়।

## WebSockets-এর Infrastructure প্রভাব

- **Sticky session প্রয়োজন:** Load balancer-কে connection-এর পুরো জীবনকাল ধরে একটা client-কে একই backend server-এ route করতে হবে (connection নিজেই stateful)।
- **Cross-server message delivery:** যদি server A-কে server B-তে connected একটা client-কে জানাতে হয়, তাহলে সাধারণত একটা pub/sub backplane (যেমন, Redis Pub/Sub) server-গুলোর মধ্যে message relay করতে ব্যবহৃত হয়।
- **Scaling:** Stateless HTTP request-response traffic-এর তুলনায় autoscale করা কঠিন; connection দীর্ঘস্থায়ী এবং তাদের সময়কাল জুড়ে server resource খরচ করে।
- **Caching/CDN:** সাধারণ HTTP caching থেকে উপকৃত হয় না কারণ এটা একটা পৃথক request-response চক্র নয়।

## কখন কোনটা ব্যবহার করবেন

| প্রয়োজন | প্রস্তাবিত টেকনিক |
|------|--------------------------|
| Server মাঝেমধ্যে update push করে, client-এর একই channel-এ ফেরত data পাঠানোর দরকার নেই | Server-Sent Events |
| সত্যিকার দুই-দিকমুখী, low-latency, high-frequency interaction | WebSockets |
| দুর্বল WebSocket/SSE সাপোর্ট আছে এমন পরিবেশ/proxy, সহজ fallback | Long polling |
| কম ঘনঘন চেক, efficiency-র চেয়ে সরলতা বেশি মূল্যবান (production-এ বিরল) | Short polling |

## মূল সংখ্যা / তথ্য

- WebSocket upgrade handshake HTTP status `101 Switching Protocols` দিয়ে সম্পন্ন হয়।
- SSE MIME type `text/event-stream` ব্যবহার করে।
- Browser-এর built-in `EventSource` API বিচ্ছিন্ন হওয়া SSE stream স্বয়ংক্রিয়ভাবে reconnect করে।

## সারসংক্ষেপ

- Real-time ফিচারের জন্য HTTP-এর শুধুমাত্র-request-response প্রকৃতির চারপাশে workaround করতে হয়।
- জটিলতা এবং ক্ষমতা বাড়ে এভাবে: short polling → long polling → SSE (এক-দিকমুখী persistent) → WebSockets (দুই-দিকমুখী persistent)।
- পছন্দটা মূলত নির্ভর করে client-এর একই real-time channel-এ ফেরত data পাঠানোর দরকার আছে কিনা তার উপর।
</content>
