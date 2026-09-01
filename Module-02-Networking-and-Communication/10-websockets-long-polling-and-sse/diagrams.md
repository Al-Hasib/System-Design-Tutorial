# Diagrams: WebSockets, Long Polling & Server-Sent Events

## 1. Short Polling vs Long Polling

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    Note over C,S: Short Polling
    C->>S: GET /messages/new
    S-->>C: 200 OK (empty)
    C->>S: GET /messages/new
    S-->>C: 200 OK (empty)
    C->>S: GET /messages/new
    S-->>C: 200 OK (new message!)

    Note over C,S: Long Polling
    C->>S: GET /messages/new
    Note right of S: Server holds request open...
    S-->>C: 200 OK (new message, sent as soon as available)
    C->>S: GET /messages/new (immediately re-opened)
```
*Short polling asks repeatedly regardless of whether there's new data; long polling holds the request open and responds the instant something arrives.*

## 2. Server-Sent Events — One Persistent, One-Way Stream

```mermaid
sequenceDiagram
    participant C as Client (EventSource)
    participant S as Server

    C->>S: GET /events (Accept: text/event-stream)
    activate S
    S-->>C: event: score_update, data: {...}
    S-->>C: event: score_update, data: {...}
    S-->>C: event: score_update, data: {...}
    deactivate S
    Note over C,S: Single connection stays open; server pushes events as they occur
```
*One HTTP connection stays open indefinitely while the server streams events down to the client; the client never sends data back over this channel.*

## 3. WebSocket Handshake and Full-Duplex Communication

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: GET /chat (Upgrade: websocket, Connection: Upgrade)
    S-->>C: 101 Switching Protocols
    Note over C,S: Connection is now a persistent WebSocket
    C->>S: message: "hello"
    S->>C: message: "hi there"
    S->>C: message: "new user joined"
    C->>S: message: "got it"
```
*After the HTTP-to-WebSocket upgrade handshake, either side can send messages to the other at any time over the same persistent connection.*
