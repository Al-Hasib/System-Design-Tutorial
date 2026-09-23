# Cheat Sheet — একটি Chat Application ডিজাইন করা (WhatsApp-এর মতো)

`README.md`-এর দ্রুত-রেফারেন্স সঙ্গী। একটি interview-এর ঠিক আগে review করার জন্য এটি ব্যবহার করুন।

## Requirements (প্রথমে এগুলো উচ্চস্বরে বলুন)

**Functional**
- 1:1 messaging
- Group messaging (কয়েকশ member পর্যন্ত)
- Delivery + read receipts (sent / delivered / read)
- Online presence ("last seen")
- Offline message delivery (কোনো message হারানো চলবে না)
- Media sharing (image/video/voice/doc)

**Non-functional**
- Low latency (দুই ব্যবহারকারীই online থাকলে এক সেকেন্ডেরও কম)
- High availability
- At-least-once delivery + idempotency (duplicate bubbles নয়)
- Durability (একবার ACK হলে, কখনো হারাবে না)

## Capacity Numbers (উত্তরটি নয়, math-এর গঠনটি মুখস্থ করুন)

| Metric | Value |
|---|---|
| Daily Active Users (DAU) | 500 মিলিয়ন |
| প্রতি ব্যবহারকারী/দিনে গড় messages | 40 |
| মোট messages/দিন | 20 বিলিয়ন |
| গড় messages/sec | ~230,000/sec |
| Peak messages/sec (গড়ের 3 গুণ) | ~700,000/sec |
| প্রতি message-এ Metadata size | ~100 bytes |
| Metadata storage/দিন | ~2 TB/দিন |
| Metadata storage/বছর (3× replication সহ) | ~2 PB/বছর |
| Media storage | আলাদা object storage (S3-এর মতো) + CDN, metadata-র সাথে inline নয় |
| Peak concurrent WebSocket connections (DAU-র ~30%) | ~150 মিলিয়ন |
| প্রতি gateway server-এ connections | ~50,000 |
| প্রয়োজনীয় gateway servers | ~3,000 (redundancy সহ ~3,500–4,000-এ round up) |

## Architecture Summary

```
Client -> Load Balancer -> Connection Gateway (WebSocket) -> Chat Service
   Chat Service -> Message Queue (Kafka pub/sub) -> Message Store (sharded by conversation ID)
   Chat Service -> Presence Service (which gateway is user X on?)
   Chat Service -> Push Notification Service (APNs/FCM) -> offline device wake-up
```

- **Connection Gateway servers**: WebSockets terminate করে, stateful, প্রতি connection-এ sticky।
- **Chat Service**: business logic, idempotency check (client message ID দিয়ে dedupe), ACK handling।
- **Message Queue**: sender-এর gateway ভিন্ন হলে recipient-এর gateway-তে একটি message route করে (gateway ID বা connected-user channel দিয়ে pub/sub)।
- **Message Store**: sharded wide-column DB, shard key = conversation ID, append-only writes, history-র জন্য range reads।
- **Presence Service**: user ID -> gateway server ID (+ online/offline/last-seen) map করে।
- **Push Notification Service**: APNs/FCM দিয়ে offline devices জাগায়; payload একটি "doorbell", message নিজে নয়।

## মূল Decisions ও Trade-offs

| Decision | Choice | কেন |
|---|---|---|
| Transport: WebSocket vs long polling | WebSocket | প্রকৃত full-duplex server push, বারবার HTTP polling-এর চেয়ে প্রতি message-এ কম overhead; খরচ হলো memory-তে লক্ষ লক্ষ idle stateful connections ধরে রাখা |
| Message store: SQL vs NoSQL | NoSQL (wide-column, যেমন Cassandra/DynamoDB-style) | Append-heavy, partition-key (conversation ID) range reads petabyte স্কেলে; multi-row transactions বা joins দরকার নেই |
| Delivery: push vs pull | recipient online থাকলে push (gateway-র মাধ্যমে), offline থাকলে reconnect-এ pull/sync | Online থাকা অবস্থায় push low latency দেয়; reconnect-on-pull নিশ্চিত করে offline থাকাকালীন কোনো message হারায় না |
| Consistency: CP vs AP | AP (availability-favoring, eventual consistency) | Partition-এর সময় একটি failed send-এর চেয়ে সামান্য দেরিতে/ভুল order-এ পৌঁছানো message ব্যবহারকারীরা পছন্দ করেন; durability non-negotiable, perfect real-time ordering নয় |
| Delivery guarantee | At-least-once + idempotency key (client-generated message ID) | একটি unreliable mobile network জুড়ে প্রকৃত exactly-once অবাস্তব; server-এ dedupe করা ব্যবহারকারীর জন্য এটি approximate করে |
| Cross-gateway routing | Message Queue / pub-sub | Sender এবং recipient সাধারণত ভিন্ন gateway servers-এ pinned থাকে; queue "কে পাঠিয়েছে" থেকে "কে live connection ধরে রেখেছে"-কে decouple করে এবং gateway crashes-এর বিরুদ্ধে buffer করে |
| Messages-এর জন্য Sharding key | Conversation ID | একটি conversation-এর messages fast, ordered history reads-এর জন্য একসাথে রাখে |
| Storage tiering | Hot sharded store (সাম্প্রতিক) + cold/archival object storage (পুরনো) | বছরে ~2 PB storage বৃদ্ধির খরচ নিয়ন্ত্রণ করে |
