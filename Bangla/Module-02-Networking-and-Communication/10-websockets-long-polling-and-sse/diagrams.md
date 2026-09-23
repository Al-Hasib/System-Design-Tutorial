# Diagrams: WebSockets, Long Polling & Server-Sent Events

## ১. Short Polling বনাম Long Polling

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

*Short polling নতুন data আছে কিনা তা না দেখেই বারবার জিজ্ঞেস করে; long polling request খোলা রাখে এবং কিছু আসা মাত্রই জবাব দেয়।*

## ২. Server-Sent Events — একটা Persistent, এক-দিকমুখী Stream

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
    Note over C,S: Single connection stays open — server pushes events as they occur
```

*একটা HTTP connection অনির্দিষ্টকালের জন্য খোলা থাকে যতক্ষণ server client-এর কাছে event stream করে; client এই channel-এ কখনো ফেরত data পাঠায় না।*

## ৩. WebSocket Handshake এবং Full-Duplex Communication

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

*HTTP থেকে WebSocket-এ upgrade handshake হওয়ার পর, একই persistent connection-এর উপর দিয়ে যে-কোনো পক্ষ যে-কোনো সময় অপর পক্ষকে message পাঠাতে পারে।*
</content>
