# Cheat Sheet — Design a Chat Application (like WhatsApp)

Quick-reference companion to `README.md`. Use this to review right before an interview.

## Requirements (say these out loud first)

**Functional**
- 1:1 messaging
- Group messaging (up to a few hundred members)
- Delivery + read receipts (sent / delivered / read)
- Online presence ("last seen")
- Offline message delivery (no lost messages)
- Media sharing (image/video/voice/doc)

**Non-functional**
- Low latency (sub-second when both users online)
- High availability
- At-least-once delivery + idempotency (no duplicate bubbles)
- Durability (once ACKed, never lost)

## Capacity Numbers (memorize the shape of the math, not just the answer)

| Metric | Value |
|---|---|
| Daily Active Users (DAU) | 500 million |
| Avg messages/user/day | 40 |
| Total messages/day | 20 billion |
| Avg messages/sec | ~230,000/sec |
| Peak messages/sec (3x avg) | ~700,000/sec |
| Metadata size/message | ~100 bytes |
| Metadata storage/day | ~2 TB/day |
| Metadata storage/year (with 3x replication) | ~2 PB/year |
| Media storage | Separate object storage (S3-like) + CDN, not inline with metadata |
| Peak concurrent WebSocket connections (~30% of DAU) | ~150 million |
| Connections per gateway server | ~50,000 |
| Gateway servers needed | ~3,000 (round up to ~3,500–4,000 with redundancy) |

## Architecture Summary

```
Client -> Load Balancer -> Connection Gateway (WebSocket) -> Chat Service
   Chat Service -> Message Queue (Kafka pub/sub) -> Message Store (sharded by conversation ID)
   Chat Service -> Presence Service (which gateway is user X on?)
   Chat Service -> Push Notification Service (APNs/FCM) -> offline device wake-up
```

- **Connection Gateway servers**: terminate WebSockets, stateful, sticky per connection.
- **Chat Service**: business logic, idempotency check (dedupe by client message ID), ACK handling.
- **Message Queue**: routes a message from the sender's gateway to the recipient's gateway when they differ (pub/sub by gateway ID or connected-user channel).
- **Message Store**: sharded wide-column DB, shard key = conversation ID, append-only writes, range reads for history.
- **Presence Service**: maps user ID -> gateway server ID (+ online/offline/last-seen).
- **Push Notification Service**: wakes offline devices via APNs/FCM; payload is a "doorbell," not the message itself.

## Key Decisions & Trade-offs

| Decision | Choice | Why |
|---|---|---|
| Transport: WebSocket vs long polling | WebSocket | True full-duplex server push, lower overhead per message than repeated HTTP polling; cost is holding millions of idle stateful connections in memory |
| Message store: SQL vs NoSQL | NoSQL (wide-column, e.g. Cassandra/DynamoDB-style) | Append-heavy, partition-key (conversation ID) range reads at petabyte scale; don't need multi-row transactions or joins |
| Delivery: push vs pull | Push when recipient online (via gateway), pull/sync on reconnect when offline | Push gives low latency while online; pull-on-reconnect guarantees no message is lost while offline |
| Consistency: CP vs AP | AP (availability-favoring, eventual consistency) | Users prefer a message that arrives slightly late/out-of-order over a failed send during a partition; durability is non-negotiable, perfect real-time ordering is not |
| Delivery guarantee | At-least-once + idempotency key (client-generated message ID) | True exactly-once is impractical across an unreliable mobile network; dedupe on the server approximates it for the user |
| Cross-gateway routing | Message Queue / pub-sub | Sender and recipient are usually pinned to different gateway servers; queue decouples "who sent it" from "who's holding the live connection" and buffers against gateway crashes |
| Sharding key for messages | Conversation ID | Keeps a conversation's messages co-located for fast, ordered history reads |
| Storage tiering | Hot sharded store (recent) + cold/archival object storage (old) | Controls cost as storage grows ~2 PB/year |
