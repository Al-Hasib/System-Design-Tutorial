# Diagrams — একটি Chat Application ডিজাইন করা (WhatsApp-এর মতো)

## ১. সামগ্রিক Architecture

```mermaid
flowchart LR
    Client1[Client A]
    Client2[Client B]
    LB[Load Balancer]
    GW1[Connection Gateway 1<br/>WebSocket]
    GW2[Connection Gateway 2<br/>WebSocket]
    Chat[Chat Service]
    Queue[Message Queue<br/>Kafka pub/sub]
    Store[(Message Store<br/>sharded by conversation ID)]
    Presence[Presence Service]
    Push[Push Notification Service<br/>APNs / FCM]

    Client1 -- WebSocket --> LB
    Client2 -- WebSocket --> LB
    LB --> GW1
    LB --> GW2
    GW1 <--> Chat
    GW2 <--> Chat
    Chat --> Queue
    Chat --> Store
    Chat <--> Presence
    Chat --> Push
    Queue --> GW1
    Queue --> GW2
    Push -.-> Client2
```

*Clients একটি নির্দিষ্ট Connection Gateway-তে pinned persistent WebSocket connections ধরে রাখে; Chat Service messages sharded storage-এ persist করে এবং প্রতিটি message recipient-এর gateway-তে route করার জন্য, অথবা recipient offline থাকলে Push Notification Service-এ route করার জন্য Message Queue এবং Presence Service ব্যবহার করে।*

## ২. ভিন্ন Connection Servers-এ থাকা দুই ব্যবহারকারীর মধ্যে Message Delivery

```mermaid
sequenceDiagram
    participant A as User A (sender)
    participant GW1 as Connection Gateway 1
    participant Chat as Chat Service
    participant Presence as Presence Service
    participant Store as Message Store
    participant Queue as Message Queue
    participant GW2 as Connection Gateway 2
    participant B as User B (recipient)
    participant Push as Push Notification Service

    A->>GW1: Send message (client message ID, text)
    GW1->>Chat: Forward message
    Chat->>Chat: Dedupe on client message ID (idempotency)
    Chat->>Store: Persist message (shard = conversation ID)
    Chat->>Presence: Lookup gateway for User B

    alt User B is online (connected to Gateway 2)
        Presence-->>Chat: User B on Gateway 2
        Chat->>Queue: Publish message for Gateway 2
        Queue->>GW2: Deliver message
        GW2->>B: Push message over WebSocket
        B-->>GW2: Delivery ACK
        GW2-->>Chat: Update delivery status
        Chat-->>GW1: Ack to sender (delivered)
        GW1-->>A: Show "delivered"
    else User B is offline
        Presence-->>Chat: User B offline
        Chat->>Push: Trigger push notification
        Push-->>B: Silent/data push (APNs/FCM)
        Note over B: App wakes, reconnects to a gateway
        B->>GW2: Reconnect + sync missed messages
        GW2->>Store: Fetch undelivered messages for User B
        Store-->>GW2: Missed messages
        GW2-->>B: Deliver over WebSocket
        GW2-->>Chat: Update delivery status
        Chat-->>GW1: Ack to sender (delivered)
        GW1-->>A: Show "delivered"
    end
```

*Gateway 1-এ থাকা User A-র একটি message persist করা হয় এবং Message Queue-এর মাধ্যমে User B-র Gateway 2-তে route করা হয় যদি সে online থাকে, অথবা User B offline থাকলে একটি push notification এবং Message Store থেকে একটি reconnect-and-sync pull trigger করে।*
