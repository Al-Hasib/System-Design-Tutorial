# Practice & Interview Questions

**1. What three guarantees does TCP provide that UDP does not?**
Reliability (guaranteed delivery with automatic retransmission of lost packets), ordering (bytes/packets are delivered to the application in the order they were sent), and congestion control (the connection slows down when it detects network congestion).

**2. Why does a live video call typically use UDP instead of TCP?**
A late packet (an old, stale video frame) is worse than a lost one — TCP would pause delivery of everything after a lost packet until it's retransmitted, freezing the call. UDP just drops the lost frame and keeps playing, trading a brief visual glitch for uninterrupted low latency.

**3. What is the "three-way handshake," and why does it matter for latency?**
The SYN → SYN-ACK → ACK sequence TCP uses to establish a connection before any data can be sent. It costs at least one full round trip before the first byte of application data moves — which is why a fresh HTTPS connection is even slower, since it pays for a TCP handshake and then a TLS handshake on top.

**4. Why does DNS typically use UDP?**
DNS queries and responses are usually small, and UDP avoids the overhead of a TCP handshake for a single small request/response — if a query is lost, the application (resolver) just retries, which is cheaper overall than the connection setup TCP would require. DNS falls back to TCP for larger responses that need reliable, ordered delivery.

**5. What is gRPC, and what two things does it build on top of?**
gRPC is an RPC (Remote Procedure Call) framework from Google that lets clients call remote methods as if they were local functions. It's built on HTTP/2 for transport (multiplexed streams, itself running on TCP) and Protocol Buffers for serialization.

**6. Give two concrete advantages gRPC has over REST/JSON for internal service-to-service calls.**
Any two of: smaller, faster-to-serialize binary payloads (Protocol Buffers vs. JSON text); native support for streaming (client, server, or bidirectional) without bolting on WebSockets/SSE; multiplexed connections via HTTP/2 avoiding per-request connection overhead; a strongly typed service contract (.proto file) shared between client and server.

**7. Why is gRPC usually a poor choice for a public, browser-facing API?**
Browsers can't natively speak gRPC's HTTP/2 framing the way they speak plain HTTP/JSON — it needs a gRPC-Web proxy layer — and the binary Protobuf payloads aren't human-readable, making ad-hoc debugging (e.g. curling an endpoint and reading the response) much harder than with REST/JSON. Public APIs also benefit more from REST's broad tooling and caching support.

**8. What is head-of-line blocking, and which protocol in this video avoids it and why?**
Head-of-line blocking is when one lost or delayed unit of data blocks delivery of everything queued behind it, even data that already arrived. TCP suffers from this at the transport layer since it must deliver bytes in order. UDP avoids it because each datagram is independent — losing one doesn't block delivery of the next.

**9. Scenario: You're designing an online multiplayer game that needs to send player position updates 20 times per second. Which transport would you choose, and why?**
UDP — a stale position update (from a dropped and retransmitted packet) is worse than a slightly-lost one, since the next update arrives in 50ms anyway. The game can layer its own lightweight logic (e.g., always trusting the most recent update, ignoring out-of-order old ones) rather than paying TCP's retransmission and head-of-line-blocking cost.

**10. A teammate suggests replacing all of your internal microservice REST APIs with gRPC. What trade-offs would you raise before agreeing?**
Debuggability drops (binary payloads, need gRPC-specific tooling instead of curl/Postman), it adds a learning curve and `.proto` contract-management overhead, and it doesn't work directly from browsers if any internal consumer is a frontend. It's a strong fit if the internal calls are high-throughput, need streaming, or already benefit from strict typed contracts — worth confirming those conditions apply before migrating everything.

**11. True or False: HTTP/3 runs directly on top of TCP, just like HTTP/1.1 and HTTP/2.**
False. HTTP/3 runs on QUIC, which is built on top of UDP — specifically to avoid the head-of-line blocking that TCP imposes on HTTP/1.1 and HTTP/2, while QUIC reimplements reliability and ordering itself, per-stream, in user space.
