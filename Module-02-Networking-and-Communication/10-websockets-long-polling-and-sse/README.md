# WebSockets, Long Polling & Server-Sent Events

**Difficulty:** Intermediate
**Estimated length:** 12-15 min
**Prerequisites:** [06 - HTTP/HTTPS & REST APIs Explained](../06-http-https-and-rest-apis/README.md), [09 - API Gateway & Backend-for-Frontend Pattern](../09-api-gateway-and-bff-pattern/README.md)

## Learning Objectives

- Explain why plain HTTP request-response struggles with real-time features.
- Describe how short polling, long polling, Server-Sent Events, and WebSockets each work.
- Compare the four techniques on latency, overhead, and directionality.
- Understand the WebSocket handshake and why it upgrades from HTTP.
- Choose the right real-time technique for a given system design scenario.

## Script

### Hook / Intro

Everything we've covered in this module so far — HTTP, load balancers, proxies, gateways — assumes a very specific pattern: the client asks, the server answers, done. But think about a chat app. When someone sends you a message, you don't refresh the page to see it — it just appears. Think about a stock ticker, a live sports score, a collaborative document showing your teammate's cursor moving in real time. None of these fit the "client asks, server answers" model, because the server needs to push data to the client the instant something happens, without the client asking first. Today we're covering the three real techniques used to solve this in production: long polling, Server-Sent Events, and WebSockets — plus their less-useful ancestor, short polling — and exactly when to reach for each one.

### The Core Problem: HTTP Wasn't Built for This

Remember from our HTTP video: it's a request-response protocol. The server cannot spontaneously send data to the client — the client always has to initiate. So how do you get "real-time" behavior out of a protocol that fundamentally requires the client to ask first? There are a few clever workarounds, each with different tradeoffs, and understanding them in order — from the naive solution to the sophisticated one — is the best way to really get why WebSockets exist.

### Short Polling

The naive solution: have the client just ask, repeatedly, forever. Every few seconds, the client fires a `GET /messages/new` request. If there's new data, great, it gets it. If not, the server just replies with an empty result immediately. Then the client waits a fixed interval and asks again.

This works, technically, but it's wasteful in two ways. First, latency: if new data arrives right after a poll, the client won't see it until the next poll interval — you're always trading off timeliness against wasted requests. Poll every second and it's more real-time but murders your server with mostly-empty requests; poll every thirty seconds and you save resources but responses feel laggy. Second, it doesn't scale gracefully: imagine a million connected clients each polling every few seconds — that's a constant, massive stream of mostly-useless requests hitting your infrastructure. Short polling is simple to implement but almost never the right production answer for real real-time needs.

### Long Polling

Long polling improves on this cleverly, using the exact same HTTP request-response mechanism, but changing server behavior. The client sends a request, and instead of the server responding immediately with "nothing new," it *holds the connection open* — doesn't respond at all — until either new data becomes available or a timeout is reached. The moment there's something new, the server responds immediately with that data. The client then, as soon as it gets a response, immediately opens a brand new long-poll request, and the cycle repeats.

This dramatically cuts wasted requests compared to short polling — you're not constantly asking and getting empty answers — and it delivers new data close to the moment it becomes available, since the server responds as soon as it has something. The cost is that it ties up a server connection/thread for the duration of the hold, which can strain server resources at scale (though modern async server architectures handle this reasonably well), and there's still some overhead from constantly re-establishing new HTTP connections after each response.

### Server-Sent Events (SSE)

Server-Sent Events take a different approach: instead of repeatedly opening new requests, the client opens a single HTTP connection to the server, and the server keeps that connection open indefinitely, streaming new events down to the client over time as plain text in a specific format, using a `Content-Type: text/event-stream`. This is a one-directional stream — the server can keep pushing events to the client on that same long-lived connection whenever it wants, without the client needing to ask again.

SSE has some genuinely nice properties: it's built directly on plain HTTP, so it works through normal proxies, load balancers, and firewalls without special handling; the browser's built-in `EventSource` API handles automatic reconnection if the connection drops; and it's simple to implement on the server side — you're just keeping a response stream open and writing to it. Its key limitation is that it's **one-way only** — server to client. If the client needs to send data back, it needs a separate, normal HTTP request alongside the SSE stream. SSE is a great fit for things like live notification feeds, live score updates, or streaming AI-generated text token by token, where the server is the one generating a continuous stream of updates and the client is mostly just listening.

### WebSockets

WebSockets solve the problem completely differently, and more powerfully: they establish a **persistent, full-duplex** connection between client and server. Full-duplex means both sides can send messages to each other at any time, independently, over the same open connection — it's not request-response at all anymore.

Here's how it starts: the client makes a normal HTTP request but includes special headers asking to "upgrade" the connection — `Connection: Upgrade` and `Upgrade: websocket`. If the server supports it, it responds with a `101 Switching Protocols` status, and at that point the underlying TCP connection stops speaking HTTP entirely and switches to the WebSocket protocol — a lightweight framing protocol that lets either side send a message to the other at any moment, with very little overhead per message compared to a full HTTP request.

WebSockets are the right tool when you need true bidirectional, low-latency communication — chat applications, multiplayer games, collaborative editing tools, live trading platforms. The tradeoff is complexity: because the connection is stateful and long-lived, it interacts differently with the infrastructure we've covered in this module. Load balancers need "sticky" routing so a client stays connected to the same backend server for the life of the connection, since the connection itself holds state. Scaling out means you now need a way for a message arriving at server A to reach a client connected to server B — commonly solved with a pub/sub backplane like Redis, which we'll cover properly in Module 5. And WebSocket connections don't play as simply with plain HTTP caching or straightforward horizontal autoscaling as stateless request-response traffic does.

### Comparing All Four

Quick mental model: short polling is client-driven and wasteful. Long polling is client-driven but efficient, holding requests open. SSE is server-driven, one-way, and simple. WebSockets are fully bidirectional and the most powerful, but also the most complex to operate at scale. As a rule of thumb: if you only need the server pushing updates to the client, prefer SSE for its simplicity. If you need true two-way, low-latency interaction, go with WebSockets. Long polling remains a reasonable fallback for environments where WebSockets or SSE aren't well supported.

### Real-World Example

Consider a customer support chat widget. The customer typing a message and sending it is a normal, simple action — that could just be a `POST /messages` REST call using everything we covered in our HTTP video. But when the support agent replies, the customer's browser needs to see that message appear immediately, without refreshing. A chat product built for real bidirectional messaging — where both sides are constantly sending short messages back and forth — would typically use a WebSocket connection for the whole conversation. But a simpler notification feature — say, "an agent has joined the chat" or "your ticket status changed" — where the server occasionally needs to push updates to a client that isn't sending anything back on that channel, is a textbook fit for Server-Sent Events instead: simpler infrastructure, no need for sticky sessions or a full duplex protocol.

### Recap

Plain HTTP can't let a server push data on its own — the client always has to ask. Short polling repeatedly asks and is wasteful. Long polling holds the request open until there's something to say, cutting waste while staying close to real-time. Server-Sent Events open one persistent, one-directional stream from server to client, simple and built on plain HTTP. WebSockets open a persistent, full-duplex connection where either side can send anytime — the most powerful option, but the most operationally complex, requiring sticky routing and cross-server message delivery at scale.

### What's Next

That wraps up Module 2 — we've now covered how systems talk: HTTP as the foundational language, load balancers and proxies as the traffic and shielding layer, gateways and BFFs orchestrating microservices, and real-time protocols for when push beats pull. In the next module, we shift from communication to storage: we'll start with SQL vs. NoSQL, digging into how to choose the right database model for the data your system actually needs to store.

## Key Takeaways

- Plain HTTP is request-response only; the server cannot spontaneously push data without one of these workaround techniques.
- Short polling repeatedly asks the server and wastes requests; long polling holds the request open until new data exists, reducing waste while staying near real-time.
- Server-Sent Events give a simple, one-directional (server-to-client) persistent stream over plain HTTP, with built-in browser reconnection support.
- WebSockets give a persistent, full-duplex connection where both sides can send messages anytime — most powerful, but requires sticky load balancing and cross-server message delivery to scale.
- Choose based on directionality: server-only push favors SSE; true two-way, low-latency interaction favors WebSockets; long polling is a solid fallback.
