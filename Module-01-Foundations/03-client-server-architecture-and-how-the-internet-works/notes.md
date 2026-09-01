# Notes: Client-Server Architecture & How the Internet Works

## Client-Server vs Peer-to-Peer

| Model | Description | Example |
|---|---|---|
| Client-Server | Clients request, servers respond; asymmetric roles | Web browsing, mobile apps, most SaaS |
| Peer-to-Peer (P2P) | Every node is both client and server | BitTorrent, some blockchain networks |

## The Request Pipeline (URL to Page)

1. **DNS Resolution** — domain name → IP address
2. **TCP Handshake** — reliable connection established (SYN, SYN-ACK, ACK)
3. **TLS Handshake** (if HTTPS) — encryption negotiated
4. **HTTP Request/Response** — actual data exchanged

## DNS (Domain Name System)

- Analogy: phone book for the internet.
- Resolution order: browser/OS cache → DNS resolver → root server → TLD server (e.g., `.com`) → authoritative server for the domain.
- Heavily cached at every level (browser, OS, ISP) to avoid repeating slow lookups.

## TCP/IP

| Layer | Responsibility |
|---|---|
| IP (Internet Protocol) | Addressing and routing packets between machines across networks |
| TCP (Transmission Control Protocol) | Reliability: ordering, retransmission of lost packets, connection state |

**Three-way handshake:**
1. Client → Server: SYN
2. Server → Client: SYN-ACK
3. Client → Server: ACK
(Connection now established; data can flow)

## HTTP

| Component | Request | Response |
|---|---|---|
| Method | GET, POST, PUT, DELETE, etc. | — |
| Path | e.g. `/search` | — |
| Headers | metadata (auth, content-type) | metadata (content-type, caching) |
| Body | data sent (e.g., form/JSON) | data returned (HTML/JSON/etc.) |
| Status Code | — | e.g., 200 OK, 404 Not Found, 500 Server Error |

- **HTTPS** = HTTP + TLS (Transport Layer Security) → encrypts traffic, verifies server identity.

## Quick Revision Bullets

- Client-server: asymmetric request/response roles; P2P: symmetric roles.
- DNS turns names into IP addresses via a hierarchical, cached lookup chain.
- TCP guarantees ordered, reliable delivery via a three-way handshake; IP handles addressing/routing.
- HTTP defines the request/response format; HTTPS adds encryption via TLS.
- Full pipeline: DNS → TCP handshake → (TLS handshake) → HTTP request/response.
