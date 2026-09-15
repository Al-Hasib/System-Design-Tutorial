# Why This Topic Matters: Transport Protocols — TCP vs UDP & Where gRPC Fits

> **In one sentence:** TCP's guarantees are so convenient that most engineers forget they have a price, and that price — head-of-line blocking and retransmission delay — is exactly what makes a video call stutter or a game feel laggy.

## The World Before This Idea

Almost everything an application developer touches runs on TCP, and TCP is genuinely excellent: bytes arrive, in order, without duplication, or the connection fails loudly. You never think about it.

Then you build something real-time. A voice call over TCP: one packet is lost, so TCP retransmits it and **holds every subsequent packet** until the missing one arrives, because it must deliver in order. The listener hears silence, then a burst of stale audio. The retransmitted packet contained 20 milliseconds of speech from half a second ago — it is worthless, and waiting for it has ruined the next half second too.

That is head-of-line blocking. TCP's reliability guarantee, which is exactly what you want for a file transfer, is actively harmful when data has an expiry date.

## The Problems It Solves

### 1. Real-time data ruined by reliability
**What you see:** Video calls that freeze and then fast-forward. Multiplayer games where your position snaps backward. Live streams that buffer instead of dropping a frame.

**Why it happens:** TCP will not deliver packet N+1 until packet N has arrived, even if N+1 is already sitting in the buffer and N is now irrelevant.

**How UDP solves it:** UDP delivers whatever arrives, immediately, with no ordering and no retransmission. A lost audio packet is simply a 20 ms glitch, and the next packet plays on time. For data whose value decays faster than a retransmission round trip, dropping is strictly better than waiting — which is why WebRTC, game netcode, VoIP, and live streaming all run on UDP.

### 2. Connection setup latency on short-lived requests
**What you see:** A tiny API call takes 300 ms, and almost all of it is setup — TCP handshake, then TLS handshake, before a single byte of your request moves.

**Why it happens:** TCP needs a round trip to establish; TLS needs one or two more. For a cross-region call, that is three round trips before the request begins.

**How this knowledge solves it:** It explains why connection reuse (keep-alive, connection pooling, HTTP/2 multiplexing) matters so much, and why QUIC — which runs over UDP and merges transport and crypto handshakes into one round trip — was worth inventing. Understanding the cost is what makes the mitigations obvious rather than cargo-culted.

### 3. Per-request overhead in high-volume internal traffic
**What you see:** Internal services spending real CPU on parsing text headers and serializing JSON, millions of times a second.

**Why it happens:** HTTP/1.1 with JSON is verbose and requires one request per connection at a time, so clients open many connections and pay setup costs repeatedly.

**How gRPC solves it:** It runs over HTTP/2, which multiplexes many concurrent streams over a single connection — no per-request setup, no head-of-line blocking at the HTTP layer, and compressed binary headers. Combined with Protocol Buffers, payloads are a fraction of the size and serialization is dramatically cheaper. For service-to-service traffic at volume, this is a substantial, measurable win.

### 4. Streaming RPC patterns that request/response cannot express
**What you see:** You need to push a continuous series of updates from server to client, or upload a stream of chunks, and you are building it awkwardly on top of polling or chunked responses.

**Why it happens:** Classic HTTP request/response is one message in, one message out.

**How gRPC solves it:** It offers server streaming, client streaming, and bidirectional streaming as first-class call types, generated into typed client and server code. This is a genuinely different capability, not just a performance optimization.

## The Price You Pay

- **UDP makes reliability your problem.** If you need any ordering, deduplication, congestion control, or retransmission, you must implement it yourself — and doing it badly is worse than TCP. This is why you should use a protocol built on UDP (QUIC, WebRTC, an established game networking library) rather than writing your own.
- **UDP is often blocked.** Corporate firewalls and some networks restrict UDP, so real deployments need TCP fallback paths.
- **gRPC is awkward at the edge.** Browsers cannot speak gRPC natively (it needs gRPC-Web plus a proxy), payloads are not human-readable, and you cannot debug with `curl` or read a request in a log. This is why the standard pattern is REST/JSON at the public edge and gRPC internally.
- **Binary contracts require tooling discipline.** Protobuf schemas must be shared, versioned, and evolved compatibly. Field numbers are permanent. Without a schema registry or a shared repository, this gets painful.
- **HTTP/2 moves head-of-line blocking rather than removing it.** Streams are multiplexed at the HTTP layer, but they still share one TCP connection, so a lost TCP segment stalls all of them. HTTP/3 over QUIC is what actually fixes this.

## When You Need It — and When You Don't

| Use TCP when | Use UDP when |
|---|---|
| Every byte must arrive (files, APIs, databases) | Data expires quickly (audio, video, game state) |
| Order matters | Low latency beats completeness |
| You want the OS to handle reliability | You will implement exactly the reliability you need |

| Use gRPC when | Use REST/JSON when |
|---|---|
| Internal service-to-service at high volume | Public APIs and third-party integrations |
| You want generated, type-checked contracts | Browser clients and easy debuggability matter |
| You need streaming RPC | Caching by HTTP intermediaries matters |
| Polyglot services need one contract definition | Simplicity outweighs efficiency |

## Why This Shows Up in Interviews

Video streaming, voice/video calling, gaming, and real-time location tracking are all standard design prompts, and each hinges on choosing UDP for the media path with a clear justification. On the service side, "how do your services talk to each other?" invites gRPC as an answer — but only a well-reasoned one: high internal volume, typed contracts, streaming needs, while keeping REST at the edge. Explaining head-of-line blocking concretely is one of the cleaner ways to demonstrate you understand the layer beneath your framework.

## How It Connects

This is the layer underneath **HTTP and REST** (topic 6) and **WebSockets** (topic 10) — WebSockets run on TCP, which is precisely why WebRTC exists for media. **Protocol Buffers** (topic 35) are gRPC's payload format. **TLS** (topic 36) sits between transport and application. **Web server internals** (topic 34) covers how a server handles thousands of these connections at once, and the choice matters most in the **video streaming** (topic 53) and **ride-sharing** (topic 54) case studies.

**Next:** [Web Server Internals](../34-web-server-internals-concurrency-and-content-serving/why.md) — what actually happens on the server when ten thousand of these connections arrive at once.
