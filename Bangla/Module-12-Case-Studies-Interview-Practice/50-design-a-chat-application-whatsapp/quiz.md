# Follow-Up Interview Questions — একটি Chat Application ডিজাইন করা (WhatsApp-এর মতো)

ভিডিওটি দেখার পর আপনার বোঝাপড়া পরীক্ষা করতে এগুলো ব্যবহার করুন। Model answer পড়ার আগে নিজে উত্তর দেওয়ার চেষ্টা করুন।

**১. একই সাথে একাধিক sender যখন message পাঠাচ্ছে, তখন একটি group chat-এর মধ্যে message ordering কীভাবে নিশ্চিত করবেন?**
Write করার সময় প্রতি conversation-এ একটি monotonically increasing sequence number assign করুন (যেমন, যে shard/partition সেই conversation ID-র মালিক, সেটি দ্বারা generate করা হয়), ভিন্ন servers জুড়ে wall-clock timestamps-এর উপর নির্ভর করার বদলে। একটি conversation-এর সব messages একই shard দিয়ে route হয়, তাই sequence number সেখানে atomically assign করা যায়। Clients messages-কে arrival time নয়, sequence number অনুযায়ী sorted করে render করে। সব conversations জুড়ে perfect global ordering প্রয়োজনও নয়, চেষ্টাও করা হয় না — শুধু per-conversation ordering ব্যবহারকারীদের কাছে গুরুত্বপূর্ণ।

**২. যে ব্যবহারকারী 500টি ভিন্ন group-এর member, তাকে কীভাবে handle করবেন?**
মূল ঝুঁকি হলো presence/routing overhead এবং অনেকগুলো group একসাথে active থাকলে fan-out cost। প্রতিটি message-এ member lists duplicate করার বদলে, group membership একটি আলাদা, cacheable membership service-এ (group ID দিয়ে keyed) store করুন। Fan-out-এর জন্য, ব্যবহারকারীর actively-open groups-এর জন্য fan-out-on-write ব্যবহার করুন এবং বড় বা কদাচিৎ খোলা হয় এমন groups-এর জন্য fan-out-on-read বিবেচনা করুন (unread messages-এর feed চাহিদা অনুযায়ী compute করুন), যাতে ব্যবহারকারী actively দেখছে না এমন প্রতিটি message-এর 500টি কপি write করা এড়ানো যায়।

**৩. লক্ষ লক্ষ concurrent WebSocket connections-এ কীভাবে scale করবেন?**
Stateful Connection Gateway servers-কে horizontally scale করুন (event-loop based, যেমন Netty/Node/Go), প্রতিটিতে সীমিত সংখ্যক connections থাকে (আমাদের estimate-এ ~50,000), এবং একটি load balancer এগুলোর সামনে থেকে fleet জুড়ে নতুন connections বিতরণ করে। User ID-কে সেই নির্দিষ্ট gateway server-এ map করে এমন একটি Presence Service বজায় রাখুন যা তাদের connection ধরে রেখেছে, যাতে অন্য servers জানে কোথায় message route করতে হবে। Deploys/restarts-এ connection draining ব্যবহার করুন যাতে clients gracefully reconnect করে, messages drop না হয়ে।

**৪. Server design-এ end-to-end encryption (Signal Protocol-এর মতো)-এর প্রভাব কী?**
Server আর message content পড়তে পারে না, তাই এটি content-based features (search, spam filtering, content moderation) server-side করতে পারে না — সেগুলো হয় client-এ চলে যায় নয়তো বাদ দেওয়া হয়। Server-এর কাজ হয়ে দাঁড়ায় শুধুমাত্র opaque encrypted blobs এবং delivery-র জন্য প্রয়োজনীয় metadata (sender, recipient, timestamp) route এবং store করা। Key exchange এবং per-device key management (নিচে multi-device sync দেখুন) একটি first-class server responsibility হয়ে ওঠে যদিও message content opaque থাকে।

**৫. Multi-device sync (একজন ব্যবহারকারী একই সাথে phone, web, এবং desktop-এ logged in) কীভাবে handle করবেন?**
প্রতিটি device-কে Presence Service-এ একই user ID-র অধীনে registered একটি নিজস্ব "connection" হিসেবে বিবেচনা করুন, যাতে একজন ব্যবহারকারীর একসাথে একাধিক active gateway connections থাকতে পারে। Send-এর সময়, recipient-এর সব active devices-এ fan out করুন, এবং sender-এর history sync থাকার জন্য sender-এর অন্য devices-এও message echo করুন। End-to-end encrypted messages-এর জন্য, এর মানে প্রতিটি recipient device-এর public key দিয়ে আলাদাভাবে message encrypt করা (Signal Protocol যেভাবে করে), যা প্রতি message-এ fan-out cost বাড়িয়ে দেয়।

**৬. এখানে at-least-once এবং exactly-once delivery-র মধ্যে পার্থক্য কী, এবং এই design কোনটি ব্যবহার করে?**
At-least-once মানে একটি message একাধিকবার deliver/process হতে পারে (যেমন, একটি timed-out ACK-এর পর client retry-র কারণে); exactly-once মানে এটি ঠিক একবার process হয়, যা একটি unreliable network জুড়ে end-to-end guarantee করা ভারী coordination ছাড়া অত্যন্ত কঠিন। এই design transport level-এ at-least-once delivery ব্যবহার করে, একটি client-generated idempotency key (message ID)-র সাথে মিলিয়ে যার উপর Chat Service deduplicate করে — distributed transactions-এর খরচ ছাড়াই ব্যবহারকারীদের কার্যত-exactly-once experience দেয় (duplicate bubbles নেই)।

**৭. একটি message "read" হয়েছে তা system কীভাবে জানে, এবং স্কেলে এটি কতটা ব্যয়বহুল?**
Message দেখানো হলে recipient-এর client একটি read-receipt event (message ID + read timestamp) তার gateway-র মাধ্যমে ফেরত পাঠায়। এটি নিজস্ব একটি lightweight, high-frequency write path — দৈনিক 20B messages-এ, read receipts প্রায় metadata write volume দ্বিগুণ করে দেয়, তাই এগুলো সাধারণত client-side batch/debounce করা হয় (যেমন, প্রতি message-এর বদলে প্রতি conversation-open-এ একটি receipt) প্রতিটি message-এর জন্য আলাদাভাবে পাঠানোর বদলে।

**৮. Message history-র জন্য একটি relational database-এর বদলে NoSQL/wide-column store কেন বেছে নেবেন?**
Workload হলো append-only writes এবং conversation ID দিয়ে keyed range reads — conversations জুড়ে joins বা multi-row ACID transactions-এর প্রয়োজন নেই। Wide-column stores (Cassandra, DynamoDB-style) একটি relational database-এর চেয়ে অনেক সহজে partition key দিয়ে writes horizontally scale করে, যেখানে বছরে ~2 PB metadata-এ পৌঁছাতে ভারী manual sharding দরকার হতো। Trade-off হলো দুর্বল cross-row consistency guarantees, যা design-এর Step 4-এর AP (availability-favoring) অবস্থান বিবেচনায় গ্রহণযোগ্য।

**৯. Delivery-র মাঝামাঝি Message Queue বা একটি Connection Gateway crash করলে কী হয়?**
যেহেতু message queue-তে publish করার আগে (বা একই সাথে atomically) Message Store-এ persist করা হয়, একটি crash এটি হারায় না — durability storage থেকে আসে, মুহূর্তে queue reliable কিনা তা থেকে নয়। একটি gateway crash করলে, এর connections drop হয়, সেই gateway-র জন্য Presence Service entries stale হয়ে যায় এবং clean up করা হয় (যেমন, heartbeat timeout দিয়ে), এবং affected clients একটি healthy gateway-তে reconnect করে; crashed gateway-র জন্য queue করা যেকোনো messages redeliver করা হয় যখন recipient-এর নতুন gateway Presence-এ register করে।

**১০. Encryption/scale model না ভেঙে message search কীভাবে support করবেন?**
End-to-end encryption না ভেঙে encrypted content-এর উপর server-side full-text search সম্ভব নয়, তাই search সাধারণত client-side করা হয় locally-decrypted message cache-এর উপর (এই কারণেই chat apps একটি local on-device database রাখে)। Server-assisted search প্রয়োজন হলে (যেমন, unencrypted enterprise chat variants-এর জন্য), decrypted content থেকে server-এ একটি আলাদা search index (Elasticsearch-style) তৈরি করা হয়, এটি বুঝে যে এটি E2E guarantee দুর্বল করে দেয়।

**১১. প্রায় প্রতিটি message send-এ query করা হয় বলে, Presence Service overload হওয়া কীভাবে এড়াবেন?**
Presence data-কে আক্রমণাত্মকভাবে cache করুন (যেমন, caching module অনুযায়ী Redis-এ) সংক্ষিপ্ত TTLs সহ, কারণ "এই ব্যবহারকারী কি online এবং কোন gateway-তে" message volume-এর তুলনায় তুলনামূলকভাবে কম ঘন ঘন পরিবর্তিত হয়। Polling-এর বদলে connect/disconnect events-এ presence update করুন, এবং একটি cache miss-কে "ধরে নিন offline, push ব্যবহার করুন" হিসেবে বিবেচনা করুন, একটি ধীর lookup-এ send path block করার বদলে — perfect presence accuracy-র চেয়ে availability এবং low latency-কে প্রাধান্য দিয়ে।

**১২. সময়ের সাথে storage-এর বৃদ্ধি কীভাবে estimate এবং নিয়ন্ত্রণ করবেন, এবং আপনার archival strategy কী?**
বছরে ~2 PB metadata estimate-এর বিপরীতে বৃদ্ধি track করুন (আলাদাভাবে track করা media storage সহ) এবং একটি tiering policy সেট করুন: সাম্প্রতিক messages (যেমন, শেষ 30–90 দিন) hot, sharded, low-latency store-এ থাকে; পুরনো messages conversation ID এবং date range দিয়ে indexed সস্তা cold/archival object storage-এ সরানো হয়, ব্যবহারকারী যথেষ্ট দূরে scroll back করলে on-demand fetch করা হয়। এটি hot store-এর working set-কে — এবং তাই এর cost ও latency-কে — bounded রাখে যদিও total historical volume বাড়তেই থাকে।
