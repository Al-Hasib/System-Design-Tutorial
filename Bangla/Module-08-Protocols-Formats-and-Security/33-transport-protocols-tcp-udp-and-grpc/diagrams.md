# Diagrams: Transport Protocols — TCP vs UDP এবং gRPC

## ১. TCP Three-Way Handshake, তারপর Data Transfer

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: SYN
    S-->>C: SYN-ACK
    C->>S: ACK
    Note over C,S: Connection established
    C->>S: Data (segment 1)
    S-->>C: ACK (segment 1)
    C->>S: Data (segment 2) -- lost in transit
    S-->>C: (no ACK received)
    C->>S: Retransmit segment 2
    S-->>C: ACK (segment 2)
```
*কোনো data প্রবাহিত হওয়ার আগে TCP-এর একটি handshake দরকার, এবং প্রতিটি segment acknowledge করা হয় — একটি হারিয়ে যাওয়া segment স্বয়ংক্রিয়ভাবে retransmit হয়, যা reliable, ordered delivery নিশ্চিত করে।*

## ২. একই কাজের জন্য TCP vs UDP

```mermaid
flowchart TB
    subgraph TCP["TCP: Video File Download"]
        A1[Client] -->|Handshake| B1[Server]
        B1 -->|Ordered, reliable chunks| A1
        A1 -->|"Every byte guaranteed to arrive intact"| A1
    end

    subgraph UDP["UDP: Live Video Call"]
        A2[Client] -->|No handshake| B2[Server/Peer]
        B2 -->|Best-effort frames| A2
        A2 -->|"Dropped frame = skip it, keep playing live"| A2
    end
```
*একটি file download-এর জন্য প্রতিটি byte অক্ষত থাকা দরকার, তাই TCP-এর retransmission-এর খরচটা এখানে সার্থক। একটি live call-এর জন্য completeness-এর চেয়ে freshness বেশি দরকার, তাই UDP-এর "drop করো এবং এগিয়ে যাও" মডেলই জেতে।*

## ৩. Stack-এ gRPC কোথায় বসে

```mermaid
flowchart TD
    App[Application Code\nStubs generated from .proto] --> GRPC[gRPC Framework]
    GRPC --> ProtoBuf[Protocol Buffers\nBinary Serialization]
    GRPC --> H2[HTTP/2\nMultiplexed streams]
    H2 --> TCP[TCP\nReliable byte stream]
    TCP --> IP[IP\nPacket routing]

    REST[REST API\nJSON over HTTP/1.1 or 2] --> H1[HTTP]
    H1 --> TCP
```
*gRPC হলো HTTP/2-এর (যা নিজেই TCP-এর উপর) উপর একটি layer, যা JSON-এর বদলে Protocol Buffers ব্যবহার করে এবং HTTP/2-এর multiplexed streaming পায় — যেখানে REST সাধারণত plain HTTP-এর উপর JSON-এই থেকে যায়।*
