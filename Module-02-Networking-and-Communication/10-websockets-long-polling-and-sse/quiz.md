# Practice & Interview Questions

**1. Why can't plain HTTP support a server pushing data to a client on its own?**
HTTP is fundamentally a request-response protocol — the client must always initiate a request before the server can respond. There's no built-in mechanism for the server to spontaneously send data without the client asking first, which is why workaround techniques (polling, SSE, WebSockets) exist.

**2. What's the main drawback of short polling at scale?**
It generates a constant stream of requests from every client at fixed intervals, most of which return no new data — wasteful of server and network resources, and it also introduces latency bounded by the polling interval since new data isn't seen until the next poll.

**3. How does long polling improve on short polling?**
Instead of responding immediately with "nothing new," the server holds the request open until new data actually becomes available (or a timeout occurs), then responds immediately. This cuts down wasted empty requests while delivering data close to the moment it's available.

**4. What is Server-Sent Events (SSE), and what is its key limitation?**
SSE is a technique where the client opens a single persistent HTTP connection and the server streams events to it over time as plain text (`text/event-stream`). Its key limitation is that it's one-directional — server to client only; the client cannot send data back over that same stream.

**5. What does "full-duplex" mean in the context of WebSockets, and why does it matter?**
Full-duplex means both the client and server can send messages to each other independently and simultaneously over the same open connection. It matters because it enables true two-way, low-latency interaction (like chat or multiplayer games) that request-response-based techniques can't efficiently provide.

**6. Describe the WebSocket handshake at a high level.**
The client sends a normal HTTP request with `Upgrade: websocket` and `Connection: Upgrade` headers. If the server supports it, it responds with `101 Switching Protocols`, and from that point the TCP connection switches from HTTP semantics to the lightweight WebSocket framing protocol, remaining open for bidirectional messages.

**7. Why do WebSocket connections require "sticky" load balancing?**
Because a WebSocket connection is stateful and long-lived — once established, all messages for that session must go to the same backend server that holds the connection. A load balancer must route the client to that same server for the life of the connection, rather than distributing each message independently.

**8. If server A needs to deliver a message to a client that's connected via WebSocket to server B, how is that typically solved?**
Through a pub/sub backplane (e.g., Redis Pub/Sub or a message broker) that lets server A publish the message, which server B subscribes to and then forwards to its locally connected client — since a plain load-balanced setup gives server A no direct way to reach a connection held by server B.

**9. You're building a live sports score notification feature where users only receive updates and never send data back on that channel. Which technique would you choose, and why?**
Server-Sent Events — it's simpler to implement and operate than WebSockets (no need for sticky sessions or a full-duplex protocol), it works over plain HTTP through normal infrastructure, and the directionality (server-to-client only) exactly matches the requirement.

**10. You're building a real-time multiplayer game where both the client and server need to send frequent, low-latency updates to each other. Which technique fits best, and why?**
WebSockets — the game needs true bidirectional communication with minimal per-message overhead and low latency, which only a full-duplex, persistent connection can provide efficiently; SSE and long polling are one-directional or higher-latency and wouldn't fit this pattern well.

**11. What is one advantage SSE has over WebSockets purely from an infrastructure/operations standpoint?**
SSE is built directly on plain HTTP, so it passes through existing proxies, load balancers, and firewalls without special handling, and browsers handle automatic reconnection via `EventSource` — no sticky session requirement or custom protocol handling needed, unlike WebSockets.

**12. In an interview, how would you justify long polling as a fallback technique rather than always defaulting to WebSockets?**
Long polling works in environments or intermediary infrastructure (older proxies, corporate networks, certain restrictive firewalls) that may not fully support WebSocket upgrades, and it's simpler to implement using standard request-response HTTP semantics — a reasonable degrade-gracefully option when WebSocket connectivity can't be guaranteed for all clients.
