# নোটস: Client-Server Architecture & How the Internet Works

## Client-Server বনাম Peer-to-Peer

| Model | বর্ণনা | উদাহরণ |
|---|---|---|
| Client-Server | Client-রা request করে, server-রা সাড়া দেয়; অসম (asymmetric) ভূমিকা | Web browsing, mobile app, বেশিরভাগ SaaS |
| Peer-to-Peer (P2P) | প্রতিটি node-ই client এবং server উভয়ই | BitTorrent, কিছু blockchain network |

## Request Pipeline (URL থেকে Page)

1. **DNS Resolution** — domain name → IP address
2. **TCP Handshake** — নির্ভরযোগ্য connection স্থাপিত হয় (SYN, SYN-ACK, ACK)
3. **TLS Handshake** (HTTPS হলে) — encryption নিয়ে সমঝোতা হয়
4. **HTTP Request/Response** — প্রকৃত data বিনিময় হয়

## DNS (Domain Name System)

- Analogy: internet-এর জন্য একটি phone book।
- Resolution-এর ক্রম: browser/OS cache → DNS resolver → root server → TLD server (যেমন, `.com`) → domain-এর জন্য authoritative server।
- ধীর lookup বারবার এড়াতে প্রতিটি level-এ (browser, OS, ISP) ব্যাপকভাবে cache করা হয়।

## TCP/IP

| Layer | দায়িত্ব |
|---|---|
| IP (Internet Protocol) | network জুড়ে machine-এর মধ্যে packet-এর addressing এবং routing |
| TCP (Transmission Control Protocol) | Reliability: ordering, হারিয়ে যাওয়া packet-এর retransmission, connection state |

**Three-way handshake:**
1. Client → Server: SYN
2. Server → Client: SYN-ACK
3. Client → Server: ACK
(এখন connection established; data প্রবাহিত হতে পারে)

## HTTP

| উপাদান | Request | Response |
|---|---|---|
| Method | GET, POST, PUT, DELETE, ইত্যাদি | — |
| Path | যেমন `/search` | — |
| Headers | metadata (auth, content-type) | metadata (content-type, caching) |
| Body | পাঠানো data (যেমন, form/JSON) | ফেরত আসা data (HTML/JSON/ইত্যাদি) |
| Status Code | — | যেমন, 200 OK, 404 Not Found, 500 Server Error |

- **HTTPS** = HTTP + TLS (Transport Layer Security) → traffic encrypt করে, server-এর identity যাচাই করে।

## দ্রুত পুনরালোচনার বুলেট

- Client-server: অসম request/response ভূমিকা; P2P: সমান ভূমিকা।
- DNS একটি hierarchical, cached lookup chain-এর মাধ্যমে নাম-কে IP address-এ পরিণত করে।
- TCP একটি three-way handshake-এর মাধ্যমে ক্রমানুসারে, নির্ভরযোগ্য delivery নিশ্চিত করে; IP addressing/routing সামলায়।
- HTTP request/response-এর format নির্ধারণ করে; HTTPS TLS-এর মাধ্যমে encryption যোগ করে।
- সম্পূর্ণ pipeline: DNS → TCP handshake → (TLS handshake) → HTTP request/response।
