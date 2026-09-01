# Study Notes: WebSockets, Long Polling & Server-Sent Events

## Definitions

- **Short polling:** Client repeatedly sends requests at fixed intervals to check for new data; server responds immediately (often empty).
- **Long polling:** Client sends a request; server holds it open until new data is available or a timeout occurs, then responds; client immediately re-opens a new request.
- **Server-Sent Events (SSE):** A single, long-lived HTTP connection over which the server streams events to the client one-directionally, using `Content-Type: text/event-stream`.
- **WebSocket:** A persistent, full-duplex TCP-based connection, initiated via an HTTP "Upgrade" handshake, allowing either side to send messages at any time.
- **Full-duplex:** Both parties can send and receive simultaneously and independently over the same connection.

## Comparison Table

| Technique | Directionality | Latency | Server overhead | Complexity | Built on |
|-----------|------------------|---------|--------------------|-------------|-----------|
| Short polling | Client-driven (pull) | Poor (bounded by poll interval) | High (many mostly-empty requests) | Low | Plain HTTP requests |
| Long polling | Client-driven (pull), server delays response | Good (near real-time) | Medium (holds connections open) | Medium | Plain HTTP, held-open requests |
| Server-Sent Events (SSE) | Server → Client only | Good (server pushes immediately) | Low-medium (one persistent stream per client) | Low-medium | Plain HTTP streaming (`text/event-stream`) |
| WebSocket | Full-duplex (both ways) | Excellent (minimal per-message overhead) | Medium-high (persistent stateful connections) | High | HTTP Upgrade → dedicated WebSocket protocol |

## WebSocket Handshake

1. Client sends HTTP request with `Connection: Upgrade` and `Upgrade: websocket` headers (plus `Sec-WebSocket-Key`).
2. Server responds `101 Switching Protocols` (plus `Sec-WebSocket-Accept`).
3. Connection switches from HTTP semantics to the WebSocket framing protocol.
4. Either side can now send framed messages at any time until the connection is closed.

## Infrastructure Implications of WebSockets

- **Sticky sessions required:** Load balancer must route a client to the same backend server for the life of the connection (the connection itself is stateful).
- **Cross-server message delivery:** If server A needs to notify a client connected to server B, a pub/sub backplane (e.g., Redis Pub/Sub) is typically used to relay messages between servers.
- **Scaling:** Harder to autoscale than stateless HTTP request-response traffic; connections are long-lived and consume server resources for their duration.
- **Caching/CDN:** Doesn't benefit from typical HTTP caching since it isn't a discrete request-response cycle.

## When to Use Which

| Need | Recommended technique |
|------|--------------------------|
| Server occasionally pushes updates, client doesn't need to send data back on the same channel | Server-Sent Events |
| True two-way, low-latency, high-frequency interaction | WebSockets |
| Environments/proxies with poor WebSocket/SSE support, simple fallback | Long polling |
| Infrequent checks, simplicity valued over efficiency (rare in production) | Short polling |

## Key Numbers / Facts

- WebSocket upgrade handshake completes with HTTP status `101 Switching Protocols`.
- SSE uses the MIME type `text/event-stream`.
- Browsers' built-in `EventSource` API auto-reconnects dropped SSE streams.

## Summary

- Real-time features require working around HTTP's request-response-only nature.
- Complexity and power increase from short polling → long polling → SSE (one-way persistent) → WebSockets (two-way persistent).
- Choice depends primarily on whether the client needs to send data back on the same real-time channel.
