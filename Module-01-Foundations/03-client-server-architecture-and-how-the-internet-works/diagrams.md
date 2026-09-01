# Diagrams: Client-Server Architecture & How the Internet Works

## 1. Client-Server Model

```mermaid
flowchart LR
    C1[Client: Browser] -->|Request| S[Server]
    C2[Client: Mobile App] -->|Request| S
    S -->|Response| C1
    S -->|Response| C2
```
*Caption: Multiple clients send requests to a server, which processes them and sends back responses.*

## 2. Full Request Sequence: URL to Page

```mermaid
sequenceDiagram
    participant Browser
    participant DNS as DNS Resolver
    participant Server
    Browser->>DNS: Resolve google.com
    DNS-->>Browser: IP: 142.250.190.14
    Browser->>Server: TCP SYN
    Server-->>Browser: SYN-ACK
    Browser->>Server: ACK (connection established)
    Browser->>Server: HTTP GET /
    Server-->>Browser: HTTP 200 OK (HTML)
```
*Caption: The full journey from typing a URL to receiving a page: DNS lookup, TCP handshake, then HTTP exchange.*

## 3. TCP/IP and HTTP Layering

```mermaid
flowchart TD
    HTTP["HTTP / HTTPS\n(Application Layer - request/response format)"] --> TLS["TLS (if HTTPS)\n(Encryption)"]
    TLS --> TCP["TCP\n(Transport Layer - reliability, ordering)"]
    TCP --> IP["IP\n(Network Layer - addressing, routing)"]
```
*Caption: HTTP rides on top of TCP/IP — IP handles addressing and routing, TCP adds reliability, and HTTP defines the message format.*
