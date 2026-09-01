# Diagrams: Transport Protocols — TCP vs UDP & gRPC

## 1. TCP Three-Way Handshake, Then Data Transfer

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
*TCP requires a handshake before any data flows, and every segment is acknowledged — a lost segment is automatically retransmitted, guaranteeing reliable, ordered delivery.*

## 2. TCP vs UDP for the Same Task

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
*Downloading a file needs every byte intact, so TCP's retransmission is worth the cost. A live call needs freshness more than completeness, so UDP's "drop and move on" model wins.*

## 3. Where gRPC Sits in the Stack

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
*gRPC is a layer on top of HTTP/2 (itself on top of TCP), swapping JSON for Protocol Buffers and gaining HTTP/2's multiplexed streaming — while REST typically stays with JSON over plain HTTP.*
