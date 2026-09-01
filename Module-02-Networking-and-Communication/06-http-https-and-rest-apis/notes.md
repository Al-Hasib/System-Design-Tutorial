# Study Notes: HTTP/HTTPS & REST APIs

## Definitions

- **HTTP (HyperText Transfer Protocol):** Application-layer, stateless, request-response protocol used for communication between clients and servers, built on top of TCP/IP.
- **Stateless:** The server does not retain any memory of previous requests from a client; each request must carry all context it needs (cookies, tokens, etc.).
- **TLS (Transport Layer Security):** Cryptographic protocol that authenticates the server, negotiates keys, and encrypts data in transit. HTTPS = HTTP over TLS.
- **REST (Representational State Transfer):** Architectural style where system state is modeled as resources, each identified by a URL, manipulated via standard HTTP methods.
- **Idempotent operation:** An operation that produces the same result no matter how many times it's applied.

## HTTP Methods

| Method | Purpose | Idempotent? | Safe (no side effects)? | Typical use |
|--------|---------|-------------|--------------------------|--------------|
| GET | Retrieve resource | Yes | Yes | Fetch data |
| POST | Create resource / trigger action | No | No | Create order, submit form |
| PUT | Replace resource entirely | Yes | No | Full update |
| PATCH | Partially update resource | No (typically) | No | Partial update |
| DELETE | Remove resource | Yes | No | Remove item |

## Status Code Ranges

| Range | Meaning | Common codes |
|-------|---------|---------------|
| 1xx | Informational | 100 Continue |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx | Client error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

- **401 vs 403:** 401 = not authenticated (who are you?). 403 = authenticated but not authorized (I know you, you can't do this).

## HTTP vs HTTPS

| Aspect | HTTP | HTTPS |
|--------|------|-------|
| Encryption | None (cleartext) | TLS-encrypted |
| Server identity verification | No | Yes (certificate) |
| Default port | 80 | 443 |
| Latency | Lower (no handshake) | Slightly higher (TLS handshake), mitigated by session resumption / HTTP/2+ |
| Use in production | Not acceptable for sensitive data | Standard/required |

## HTTP Protocol Version Notes

- **HTTP/1.1:** One request per connection at a time (pipelining rarely used); keep-alive reduces reconnect overhead.
- **HTTP/2:** Multiplexes multiple requests/responses over a single TCP connection; header compression (HPACK).
- **HTTP/3:** Runs over QUIC (UDP-based) instead of TCP, avoiding TCP head-of-line blocking; faster connection setup.

## REST Principles

- Resources are nouns, identified by URLs (`/users/5`), not verbs in the path (`/getUser`).
- Actions are expressed via HTTP methods, not the URL.
- Stateless: every request is self-contained.
- Cacheable where appropriate (GET responses, `Cache-Control`, `ETag`).
- Uniform interface: consistent conventions across the whole API.
- Resources can be nested to express relationships: `/users/5/orders`.

## Good REST API Design Checklist

- Plural nouns for collections: `/users`, not `/user`.
- Use query params for filtering/sorting/pagination: `/orders?status=shipped&page=2&limit=20`.
- Version the API: `/v1/users` or an `Accept`/custom header.
- Return correct status codes, not always 200 with an error field.
- Use `Idempotency-Key` header for unsafe operations (like POST) that must tolerate retries.

## Key Numbers / Facts

- Default HTTP port: 80. Default HTTPS port: 443.
- TLS handshake typically adds 1-2 network round trips (less with TLS 1.3, which supports 1-RTT and 0-RTT resumption).
- REST was formally described in Roy Fielding's 2000 PhD dissertation.

## Summary

- HTTP is the language of client-server communication: stateless, request/response, method + status code driven.
- HTTPS adds confidentiality and server authentication via TLS at the cost of a small handshake overhead.
- REST maps CRUD-style operations onto resource URLs using HTTP methods, producing predictable, cacheable, well-structured APIs.
