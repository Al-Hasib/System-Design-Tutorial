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
*A client sends a stateless HTTP request and the server replies with a status code, headers, and a body — no memory of prior requests is kept.*

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
*HTTPS performs a TLS handshake once to authenticate the server and agree on encryption keys, then all HTTP traffic in that session is encrypted.*

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
*Each step of checkout maps to a resource and an HTTP method; POST /orders carries an idempotency key so retries don't create duplicate orders.*
