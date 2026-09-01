# Design a Chat Application (like WhatsApp)

**Difficulty:** Advanced (Capstone)
**Estimated length:** 25–30 min
**Prerequisites:**
- [WebSockets, Long Polling, and SSE](../../Module-02-Networking-and-Communication/10-websockets-long-polling-and-sse/README.md)
- [Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md)
- [CAP Theorem and PACELC](../../Module-03-Databases-and-Storage/15-cap-theorem-and-pacelc/README.md)
- [Message Queues: Kafka vs RabbitMQ](../../Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/README.md)
- [Publish-Subscribe Pattern](../../Module-05-Messaging-and-Asynchronous-Systems/21-publish-subscribe-pattern/README.md)
- [Data Consistency Models and Idempotency](../../Module-06-Distributed-Systems-Concepts/29-data-consistency-models-and-idempotency/README.md)
- [Distributed Caching: Redis and Memcached](../../Module-04-Caching-and-Content-Delivery/19-distributed-caching-redis-and-memcached/README.md)

## Learning Objectives

- Run through a complete mock interview for "design a chat application" end to end: requirements, estimation, high-level design, deep dive, trade-offs.
- Estimate capacity (messages/sec, storage, concurrent connections, and gateway server count) for a chat system operating at WhatsApp-like scale.
- Explain how WebSocket connection gateways, a message queue / pub-sub layer, and sharded message storage combine to route, deliver, and persist messages reliably.
- Justify the key trade-offs a chat system makes: availability over strict consistency, at-least-once delivery plus idempotency instead of distributed transactions, and NoSQL wide-column storage over relational storage for message history.
- Anticipate and answer common interviewer follow-ups on ordering, fan-out, multi-device sync, and offline delivery.

## Script

### Hook / Intro

"Design WhatsApp" or "design a chat application" is one of the most common capstone questions in system design interviews, precisely because it forces you to combine almost everything from earlier modules into one coherent system. You need real-time bidirectional transport (WebSockets, Module 2), you need to move messages between servers that don't share memory (message queues and pub/sub, Module 5), you need to store a firehose of small writes at scale (sharding, Module 3), you need to reason about what happens when the network partitions (CAP theorem, Module 3), and you need to make sure a flaky mobile connection doesn't cause duplicate or lost messages (idempotency, Module 6). In this video we'll run it as a full mock interview: clarify requirements, size the system with real numbers, sketch the high-level architecture, deep-dive into the two or three components that actually matter, and then talk trade-offs — exactly the flow you should use in a real interview.

### Step 1: Clarify Requirements

Before drawing a single box, clarify scope out loud. This signals to the interviewer that you don't jump to solutions.

**Functional requirements:**
- One-to-one messaging between two users.
- Group messaging (up to a few hundred members per group).
- Delivery receipts (sent → delivered → read, the classic single/double/blue check marks).
- Online presence ("last seen" / online-now indicator).
- Offline message delivery — messages must be delivered when the recipient's device comes back online, not lost.
- Media sharing (images, video, voice notes, documents).

**Non-functional requirements:**
- Low latency: messages should arrive in well under a second when both users are online.
- High availability: the system should keep accepting and forwarding messages even during partial failures — chat is the kind of product where users notice a single dropped message.
- At-least-once delivery, with the client and server working together to approximate exactly-once **effective** delivery (no duplicate bubbles shown to the user) via idempotency keys.
- Durability: once the server acknowledges a message, it must not be lost, even if the recipient is offline for weeks.

Explicitly say you're deprioritizing end-to-end encryption key management and voice/video calling as separate sub-systems, and that you'll mention them briefly in trade-offs — this keeps the interview focused.

### Step 2: Capacity Estimation

Let's put real numbers on this, the way you would on a whiteboard.

- **Daily Active Users (DAU):** 500 million.
- **Average messages sent per user per day:** 40.
- **Total messages/day:** 500M × 40 = **20 billion messages/day**.
- **Average messages/sec:** 20,000,000,000 / 86,400 ≈ **230,000 messages/sec**.
- **Peak messages/sec:** chat traffic isn't flat — it spikes around evenings across time zones and during events (New Year's Eve is the classic WhatsApp example). Using a 3× peak-to-average ratio: ≈ **700,000 messages/sec at peak**.
- **Storage per message:** metadata (message ID, sender ID, conversation ID, timestamp, delivery status, short text body) averages roughly **100 bytes**. That gives 20B × 100 bytes ≈ **2 TB/day** of metadata alone, or roughly 700 TB/year before replication — with 3× replication for durability, call it **~2 PB/year** for the message metadata store. Media is **not** stored inline; it's uploaded to a separate blob store (S3-like object storage) with the message row only holding a URL/reference, because a small percentage of messages carry multi-megabyte attachments and mixing that with the hot metadata table would wreck write latency.
- **Concurrent WebSocket connections:** not all 500M DAU are online simultaneously. Assume ~30% concurrency at peak (a common rule of thumb for messaging apps): **150 million concurrent persistent connections**.
- **Connection gateway servers needed:** at roughly 50,000 concurrent connections per gateway server (a realistic ceiling for a tuned event-loop server holding idle WebSocket connections), that's 150,000,000 / 50,000 = **3,000 gateway servers**, rounded up to ~3,500–4,000 with redundancy and headroom across regions.

These numbers drive every design decision that follows: they tell us we need horizontal sharding for storage, a fleet of stateful connection gateways (not a single load-balanced stateless pool), and an asynchronous queue to decouple ingestion from delivery.

### Step 3: High-Level Design

Sketch the request path left to right:

```
Client (mobile/web)
  → Load Balancer (L4, sticky-ish routing)
    → Connection Gateway servers (WebSocket termination)
      → Chat/Message Service (business logic, validation, idempotency check)
        → Message Queue (Kafka-based pub/sub)
          → Message Store (sharded, wide-column DB) — persists every message
          → Presence Service — tracks which gateway server each user is connected to
          → Push Notification Service — wakes up offline devices via APNs/FCM
```

Walk through the flow: a client opens a persistent WebSocket connection to a **Connection Gateway** server (through a load balancer that picks a gateway and, from then on, that TCP/WebSocket connection stays pinned to that one server — this is the "stickiness" we'll revisit in Step 5). When the user sends a message, it travels over that WebSocket to the gateway, which forwards it to the **Chat Service**. The Chat Service assigns a message ID, checks the **Presence Service** to find out which gateway server (if any) the recipient is currently connected to, persists the message to the **Message Store**, and publishes an event onto the **Message Queue**. If the recipient is online, the message is routed — possibly via the queue's pub/sub layer — to the specific gateway server holding the recipient's connection, and pushed down that WebSocket in real time. If the recipient is offline, the **Push Notification Service** sends a silent/data push through APNs (iOS) or FCM (Android) so the OS wakes the app, which then reconnects and pulls the missed messages from the Message Store.

### Step 4: Deep Dive on Key Components

This is where you connect the design back to fundamentals — exactly what an interviewer wants to hear.

**4a. Real-time transport with WebSockets (Module 2).** A chat app needs full-duplex, low-latency communication, and the server needs to be able to push to the client without the client asking first — that rules out plain HTTP polling. We use WebSockets, established once per client session and held open for as long as the app is foregrounded (with heartbeats/pings to detect dead connections and free up gateway capacity). This is exactly the trade-off covered in the WebSockets/long-polling/SSE module: WebSockets give us bidirectional push at the cost of the gateway server having to hold millions of stateful, mostly-idle connections in memory — hence the 50K-connections-per-server sizing above.

**4b. Routing between gateway servers with a message queue / pub-sub (Module 5).** Because connections are pinned to specific gateway servers, the sender and recipient are very likely connected to *different* physical gateway servers. The Chat Service can't just "call" the recipient's gateway directly at scale — instead it publishes the outgoing message to a topic (Kafka, as covered in the message-queues module) keyed by the recipient's gateway server ID, or uses a pub/sub fan-out (Redis Pub/Sub or a Kafka consumer group per gateway) where each gateway subscribes to a channel for the user IDs currently connected to it. This decouples "who sent it" from "who's holding the live connection," and it also gives us a durable buffer: if a gateway server crashes mid-delivery, the message isn't lost — it's still sitting in the queue/store and gets redelivered.

**4c. Sharded storage for message history (Module 3).** With ~2 PB/year of message metadata, a single database instance is a non-starter. We shard the Message Store by **conversation ID** (for 1:1 chats, a deterministic hash of the two user IDs; for groups, the group ID) so that all messages in a given conversation land on the same shard and can be fetched in order with a single range query — this is the same conversation-locality pattern you'd use sharding any other append-heavy, read-by-partition-key workload. A wide-column store (Cassandra/HBase-style, or DynamoDB) fits well because writes are append-only and reads are almost always "give me the last N messages for this conversation."

**4d. Idempotency and delivery guarantees (Module 6).** Mobile networks drop and retry constantly, so the client-to-server hop is inherently at-least-once: the client will resend a message if it doesn't get an ACK in time. To avoid showing duplicate bubbles, the client generates a client-side message ID (UUID) before sending; the Chat Service deduplicates on that ID (an idempotency key) before persisting. Delivery status (sent/delivered/read) is tracked as a small state machine per message, with ACKs flowing back from the recipient's device through the same gateway → queue → sender's gateway path.

**4e. CAP trade-offs (Module 3).** During a network partition, we explicitly choose **availability over strict consistency** — it's far better for a user to send a message that arrives a few seconds late or slightly out of order across devices than for the send button to fail outright. We accept eventual consistency on things like read-receipt propagation and presence status, while treating message durability (never silently dropping a persisted message) as non-negotiable.

### Step 5: Bottlenecks & Trade-offs

- **Connection stickiness and scaling gateways.** Because each WebSocket is pinned to one gateway server, scaling out means adding gateway servers and rebalancing new connections onto them — but existing connections can't just be "moved." A gateway restart/deploy has to gracefully drain connections and let clients reconnect elsewhere, and the Presence Service must be updated the instant a connection moves.
- **Message ordering.** Within a single 1:1 conversation, ordering is straightforward if all messages for that conversation flow through the same shard/partition. In group chats with many concurrent senders, we typically accept "causal-ish" ordering with per-message timestamps and sequence numbers per conversation, rather than paying for a global total order.
- **Group chat fan-out.** A message to a 500-person group means 500 separate deliveries. At scale, this is done asynchronously via the queue (fan-out-on-write for active groups, sometimes fan-out-on-read for very large or inactive groups) so one slow recipient never blocks the other 499.
- **Storage growth and archival.** At ~2 PB/year of metadata plus much larger media volume, hot storage can't hold everything forever. Recent messages live in fast, sharded hot storage; older conversation history is moved to cheaper cold/archival storage (object storage with an index), consistent with typical time-based tiering strategies.
- **Push notification delivery for offline devices.** APNs/FCM delivery isn't guaranteed or instant, and payload size is limited, so pushes carry just enough to wake the app, which then pulls the real messages from the Message Store — the push path is a "doorbell," not the message transport itself.

### Recap

We clarified functional requirements (1:1 and group messaging, receipts, presence, offline delivery, media) and non-functional ones (low latency, high availability, at-least-once-with-idempotency, durability); sized the system at 20B messages/day (~230K/sec average, ~700K/sec peak), ~2 PB/year of message metadata, and 150M concurrent connections needing ~3,000+ gateway servers; walked the high-level path from client through the Connection Gateway, Chat Service, Message Queue, sharded Message Store, Presence Service, and Push Notification Service; deep-dived into WebSockets, pub/sub routing between gateways, conversation-ID sharding, and idempotent at-least-once delivery; and closed with the trade-offs around stickiness, ordering, fan-out, storage tiering, and push delivery.

### What's Next

This case study is a capstone — if any piece felt unfamiliar, go back to the linked prerequisite modules above (WebSockets, sharding, CAP theorem, message queues, pub/sub, idempotency, and caching) for the deep-dive theory behind the component we only had time to summarize here. From an interview-practice standpoint, try re-deriving this design from scratch on a blank page before checking your version against this script, then move on to the quiz in this folder to pressure-test yourself against the kind of follow-up questions a real interviewer would throw at you.

## Key Takeaways

- Chat systems are a capstone problem because they require real-time transport (WebSockets), asynchronous routing (queues/pub-sub), sharded persistence, and idempotent delivery all working together.
- Always estimate capacity with real numbers before designing — 500M DAU × 40 messages/day gives 20B messages/day, ~230K/sec average and ~700K/sec peak, which directly justifies horizontal sharding and a dedicated stateful gateway fleet.
- Persistent connections are pinned to specific gateway servers, so a queue or pub/sub layer is required to route a message from the sender's gateway to the recipient's gateway.
- Shard message history by conversation ID so that a conversation's messages stay co-located for fast, ordered range reads.
- Client-generated message IDs plus server-side deduplication turn an inherently at-least-once network into an effectively exactly-once user experience.
- Favor availability over strict consistency during partitions; treat message durability, not perfect real-time ordering, as the non-negotiable guarantee.
