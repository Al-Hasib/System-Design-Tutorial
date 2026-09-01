# Transport Protocols: TCP vs UDP & Where gRPC Fits

**Difficulty:** Intermediate
**Estimated length:** 14-18 min
**Prerequisites:** [03 - Client-Server Architecture and How the Internet Works](../../Module-01-Foundations/03-client-server-architecture-and-how-the-internet-works/README.md), [06 - HTTP/HTTPS & REST APIs Explained](../../Module-02-Networking-and-Communication/06-http-https-and-rest-apis/README.md)

## Learning Objectives

- Explain where TCP and UDP sit in the networking stack relative to HTTP.
- Describe TCP's reliability guarantees — the three-way handshake, ordering, retransmission, and congestion control — and what they cost.
- Describe UDP's "fire and forget" model and why giving up reliability buys lower latency.
- Choose the right transport protocol for a given system design scenario (web APIs, video calls, DNS, gaming, live streaming).
- Explain what gRPC is, why it's built on HTTP/2, and when to reach for it over a plain REST API.

## Script

### Hook / Intro

Back in video 6, we said HTTP is an application-layer protocol that "sits on top of TCP/IP." We treated TCP as a black box that just reliably moves bytes from one machine to another. Today we're opening that box. Understanding TCP versus UDP isn't academic trivia — it's the reason a video call can tolerate a dropped frame but a bank transfer can't tolerate a dropped byte, and it's the reason some of the highest-throughput microservice architectures in the world skip REST entirely in favor of gRPC. By the end of this video, you'll be able to justify, in an interview, exactly why a system should sit on TCP, UDP, or gRPC — and it won't be a guess.

### Where This Sits in the Stack

Recall the layering: the network moves raw packets (IP), a transport protocol decides how those packets become a reliable (or unreliable) stream of data between two endpoints, and an application-layer protocol like HTTP defines the meaning of the bytes on top of that stream. TCP and UDP are both transport-layer protocols — they're the two main choices for "how do bytes actually get from client to server," and almost everything else in this course sits on top of one of them.

### TCP: Reliability, at a Cost

TCP — Transmission Control Protocol — is connection-oriented. Before any data moves, client and server perform a three-way handshake: SYN, SYN-ACK, ACK. That handshake establishes a stateful connection both sides track, and it's the reason a fresh TCP connection always costs at least one extra round trip before you can even start a TLS handshake on top of it (remember HTTPS from video 6 — that's TLS layered on top of this TCP handshake, so a brand-new HTTPS connection pays for both).

Once connected, TCP guarantees three things: **reliability** — every byte sent is acknowledged, and lost packets are automatically retransmitted; **ordering** — bytes arrive at the application in the exact order they were sent, even if the underlying packets took different paths and arrived out of order; and **congestion control** — TCP actively slows down when it detects network congestion, to avoid making things worse for everyone sharing that network. This is why TCP is the right default for anything where correctness matters more than a few extra milliseconds: web pages, APIs, file transfers, database connections, email. If a packet drops, TCP quietly retransmits it and your application never even knows it happened.

The cost is latency and overhead. Handshakes, acknowledgments, and retransmission all take time, and head-of-line blocking means one lost packet can stall everything behind it until it's recovered — a real problem for HTTP/1.1 and HTTP/2, and part of why HTTP/3 moved to QUIC (built on UDP) specifically to escape TCP's head-of-line blocking.

### UDP: Fire and Forget

UDP — User Datagram Protocol — is connectionless. There's no handshake, no guaranteed delivery, no guaranteed ordering, and no automatic retransmission. You send a datagram, and it either arrives or it doesn't — the application has to handle that itself, if it cares at all.

That sounds strictly worse than TCP, until you consider a video call. If one video frame's packet gets lost, do you want the call to pause while TCP retransmits that one packet and holds up everything after it? Almost never — you'd rather drop that one frame and keep playing live, slightly glitchy video, than freeze the whole call waiting for a byte that's already stale by the time it arrives. This is the core insight: UDP is the right choice whenever a late packet is worse than a lost packet. Real-time voice and video (VoIP, WebRTC), competitive online gaming (where you want the freshest position update, not an old one arriving late), DNS lookups (small enough that retrying at the application level is cheaper than TCP's handshake overhead), and live streaming protocols all lean on UDP, often layering their own lightweight reliability or error-concealment on top only where it's actually needed.

### Where gRPC Fits

So where does gRPC fit into this? gRPC is a modern RPC (Remote Procedure Call) framework built by Google, and it runs on top of **HTTP/2** — which itself runs on TCP. gRPC's pitch is: instead of designing a REST API around resources and HTTP verbs, you define a **service contract** — a set of remote methods with typed request/response messages — in a `.proto` file using Protocol Buffers (we'll cover the format itself properly next video). The client then calls those methods as if they were local functions, and gRPC handles serialization, the network call, and deserialization underneath.

Three things make gRPC attractive for service-to-service communication inside a system: it uses Protocol Buffers instead of JSON, which is smaller on the wire and faster to (de)serialize; it rides HTTP/2's multiplexing, so many concurrent calls share one connection without head-of-line blocking at the HTTP layer; and it has first-class support for streaming — client-streaming, server-streaming, or full bidirectional streaming — which a request/response REST call can't express naturally. The trade-off is that gRPC is harder to call from a browser directly, less human-readable for debugging (you can't just curl it and read the response), and less familiar to third-party API consumers than a REST/JSON API. That's exactly why the common pattern is: gRPC internally between your own microservices, REST (or GraphQL) at the public-facing edge.

### Real-World Example

Picture a ride-sharing backend (something we'll design end-to-end in Module 9). The mobile app talks to the backend over HTTPS/REST — public-facing, needs to work from any browser or mobile HTTP client, benefits from caching and familiar tooling. Internally, the location service streams a driver's GPS coordinates to the matching service many times a second — a perfect fit for gRPC's server-streaming or bidirectional streaming over HTTP/2, far more efficient than opening a new REST call per coordinate update. And underneath the location updates themselves, if this were a live-tracking feature broadcasting position to nearby riders' apps in near-real-time with tolerance for an occasional missed update, UDP-based transport (or a UDP-backed protocol) would be the right instinct if latency matters more than every single update landing.

### Recap

TCP trades latency for reliability, ordering, and congestion control — the right default for anything where correctness matters more than raw speed. UDP drops all of that overhead for lower latency, which is the right trade whenever a late packet is worse than a lost one. gRPC is a high-performance RPC framework built on HTTP/2 (and therefore TCP) that uses Protocol Buffers instead of JSON, making it a strong choice for internal service-to-service communication, especially when streaming is involved.

### What's Next

We used the phrase "Protocol Buffers" several times without really defining it. Next video, we go one level up the stack from transport to look purely at message formats — comparing JSON, XML, and Protocol Buffers on the axes that actually matter in system design: bandwidth, parsing performance, human-readability, and how well each one handles schemas changing over time.

## Key Takeaways

- TCP is connection-oriented and guarantees reliability, ordering, and congestion control — at the cost of handshake latency and head-of-line blocking.
- UDP is connectionless with no delivery or ordering guarantees, trading reliability for lower latency — the right choice when a late packet is worse than a lost one.
- HTTP (and HTTPS) runs on TCP; real-time voice/video, gaming, DNS, and live streaming typically lean on UDP.
- gRPC is an RPC framework built on HTTP/2 (over TCP) that uses Protocol Buffers instead of JSON, giving it smaller payloads, multiplexed connections, and native streaming support.
- A common production pattern: REST/JSON at the public edge for compatibility and cacheability, gRPC internally between microservices for performance and streaming.
