# Study Notes: Transport Protocols — TCP vs UDP & gRPC

## Definitions

- **Transport layer:** The networking layer responsible for getting a stream (or sequence) of bytes between two endpoints; sits below application protocols like HTTP, above raw IP packet delivery.
- **TCP (Transmission Control Protocol):** Connection-oriented transport protocol providing reliable, ordered, congestion-controlled delivery.
- **UDP (User Datagram Protocol):** Connectionless transport protocol providing best-effort, unordered delivery with no automatic retransmission.
- **gRPC:** An RPC framework (by Google) built on HTTP/2, using Protocol Buffers for serialization, supporting unary and streaming calls.
- **Three-way handshake:** SYN → SYN-ACK → ACK — the setup sequence TCP uses to establish a connection before any data is sent.
- **Head-of-line blocking:** When one lost/delayed unit of data blocks delivery of everything queued behind it, even if that later data has already arrived.

## TCP vs UDP

| Aspect | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (handshake required) | Connectionless |
| Reliability | Guaranteed delivery, automatic retransmission | Best-effort, no retransmission |
| Ordering | Guaranteed | Not guaranteed |
| Congestion control | Yes, built in | None (application must handle if needed) |
| Overhead | Higher (handshake, ACKs, retransmission) | Lower (minimal header, no state) |
| Head-of-line blocking | Yes, at the TCP layer | No |
| Typical use cases | Web/HTTP, APIs, databases, file transfer, email | VoIP, video calls, gaming, DNS, live streaming |

## Where Common Protocols Sit

| Protocol | Runs On | Notes |
|---|---|---|
| HTTP/1.1, HTTP/2 | TCP | Inherit TCP's reliability but also its head-of-line blocking |
| HTTP/3 | QUIC (over UDP) | Rebuilds reliability/ordering *per stream* in user space to avoid TCP's head-of-line blocking |
| gRPC | HTTP/2 (→ TCP) | Adds Protocol Buffers + native streaming on top |
| DNS | Usually UDP (falls back to TCP for large responses) | Small requests; app-level retry is cheaper than a TCP handshake |
| VoIP / WebRTC | UDP | Tolerates lost packets far better than added latency |

## gRPC vs REST (Quick Comparison)

| Aspect | REST (JSON over HTTP/1.1 or 2) | gRPC (Protobuf over HTTP/2) |
|---|---|---|
| Payload format | JSON (text) | Protocol Buffers (binary) |
| Payload size / speed | Larger, slower to parse | Smaller, faster to (de)serialize |
| Browser support | Native (fetch/XHR) | Needs gRPC-Web proxy layer |
| Human debuggability | High (curl, readable JSON) | Low (binary, needs tooling) |
| Streaming | Awkward (WebSockets/SSE needed) | Native (client/server/bidirectional streaming) |
| Best fit | Public APIs, browser clients | Internal service-to-service calls |

## Key Numbers / Facts

- TCP handshake: 1 round trip before any application data can be sent (2 more if TLS is layered on top, though TLS 1.3 can reduce this).
- UDP header: 8 bytes, vs. TCP header: minimum 20 bytes — part of why UDP has lower per-packet overhead.
- gRPC was open-sourced by Google in 2015, built on the same HTTP/2 stack used internally at Google as "Stubby."

## Summary

- TCP buys reliability, ordering, and congestion control at the cost of handshake latency and head-of-line blocking — the right default for correctness-sensitive traffic.
- UDP strips all of that overhead for lower latency — correct when a late packet is worse than a lost one.
- gRPC layers a typed RPC contract and Protocol Buffers on top of HTTP/2 (TCP), making it a strong choice for fast, streaming-capable internal service communication, while REST/JSON remains the better fit for public, browser-facing APIs.
