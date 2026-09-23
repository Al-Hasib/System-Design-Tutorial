# ডায়াগ্রাম: Client-Server Architecture & How the Internet Works

## ১. Client-Server Model

```mermaid
flowchart LR
    C1[Client: Browser] -->|Request| S[Server]
    C2[Client: Mobile App] -->|Request| S
    S -->|Response| C1
    S -->|Response| C2
```
*Caption: একাধিক client একটি server-এ request পাঠায়, যেটি সেগুলো process করে এবং response ফেরত পাঠায়।*

## ২. সম্পূর্ণ Request Sequence: URL থেকে Page

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
*Caption: URL টাইপ করা থেকে page পাওয়া পর্যন্ত সম্পূর্ণ যাত্রা: DNS lookup, TCP handshake, তারপর HTTP বিনিময়।*

## ৩. TCP/IP এবং HTTP-এর Layering

```mermaid
flowchart TD
    HTTP["HTTP / HTTPS\n(Application Layer - request/response format)"] --> TLS["TLS (if HTTPS)\n(Encryption)"]
    TLS --> TCP["TCP\n(Transport Layer - reliability, ordering)"]
    TCP --> IP["IP\n(Network Layer - addressing, routing)"]
```
*Caption: HTTP চলে TCP/IP-এর ওপর দিয়ে — IP addressing এবং routing সামলায়, TCP reliability যোগ করে, এবং HTTP message-এর format নির্ধারণ করে।*
