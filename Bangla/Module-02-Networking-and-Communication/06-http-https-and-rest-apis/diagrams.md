# Diagrams: HTTP/HTTPS & REST APIs

## 1. Basic HTTP Request-Response Cycle

```mermaid
sequenceDiagram
    participant C as Client (Browser/App)
    participant S as Server

    C->>S: GET /users/5 HTTP/1.1 (Headers: Authorization, Accept)
    activate S
    S-->>C: 200 OK (Headers: Content-Type, Body: JSON user data)
    deactivate S
```
*একটি client একটি stateless HTTP request পাঠায় এবং server একটি status code, headers, এবং একটি body দিয়ে উত্তর দেয় — আগের requests-এর কোনো memory রাখা হয় না।*

## 2. TLS Handshake Before an HTTPS Request

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server

    C->>S: ClientHello (supported TLS versions, ciphers)
    S-->>C: ServerHello + Certificate
    C->>C: Verify certificate against trusted CA
    C->>S: Key exchange (negotiate shared symmetric key)
    Note over C,S: Secure channel established
    C->>S: Encrypted HTTP GET /users/5
    S-->>C: Encrypted 200 OK response
```
*HTTPS server-কে authenticate করতে এবং encryption keys নিয়ে সম্মত হতে একবার একটি TLS handshake সম্পন্ন করে, এরপর সেই session-এর সব HTTP traffic encrypted থাকে।*

## 3. REST Resource Model for an E-Commerce Checkout

```mermaid
flowchart LR
    A["GET /products/123"] --> B[Product Service]
    C["POST /cart/items"] --> D[Cart Service]
    E["PUT /cart/items/456"] --> D
    F["POST /orders (Idempotency-Key)"] --> G[Order Service]

    B -->|200 OK| A
    D -->|201 Created| C
    D -->|200 OK| E
    G -->|201 Created| F
```
*Checkout-এর প্রতিটি ধাপ একটি resource এবং একটি HTTP method-এর উপর map হয়; POST /orders একটি idempotency key বহন করে যাতে retries duplicate orders তৈরি না করে।*
