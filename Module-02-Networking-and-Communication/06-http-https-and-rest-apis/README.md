# HTTP/HTTPS & REST APIs Explained

**Difficulty:** Beginner/Intermediate
**Estimated length:** 12-15 min
**Prerequisites:** [03 - Client-Server Architecture and How the Internet Works](../../Module-01-Foundations/03-client-server-architecture-and-how-the-internet-works/README.md), [04 - Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md)

## Learning Objectives

- Explain what HTTP is and why it's called a "stateless, request-response" protocol.
- Describe how HTTPS secures HTTP using TLS, and why that matters for system design.
- Identify the standard HTTP methods, status code ranges, and headers, and what each is for.
- Explain REST as an architectural style and design a resource-oriented API.
- Recognize common REST API design mistakes and how to avoid them.

## Script

### Hook / Intro

Every time you load a webpage, tap "checkout" on an app, or hit an endpoint from a backend service, you're almost certainly using HTTP. It is, without exaggeration, the single most-used protocol in system design interviews and in real production systems. So today we're going to open it up completely: what HTTP actually is, what HTTPS adds on top of it, and how REST gives us a sane, consistent way to design APIs on top of both. By the end of this video you should be able to look at any API and immediately understand whether it's "RESTful," and explain confidently in an interview why a `PUT` isn't a `POST`, and why `https://` matters for more than just a padlock icon.

In the last module, we talked about client-server architecture — a client sends a request, a server sends back a response. HTTP is simply the specific language they use to have that conversation.

### What HTTP Actually Is

HTTP stands for HyperText Transfer Protocol. It's an application-layer protocol — meaning it sits on top of TCP/IP, which handles the actual movement of bytes across the network, and HTTP defines the *format* of the messages: how a client asks for something, and how a server answers.

Two properties define HTTP's core behavior. First, it's request-response: the client always initiates, the server always replies. There's no server randomly pushing data to you over plain HTTP — we'll see later in this module how WebSockets break that rule when we need real-time behavior. Second, HTTP is stateless. Each request is handled in complete isolation — the server doesn't remember your previous request unless you explicitly carry state, usually via cookies, tokens, or session identifiers sent with every request. This statelessness is actually a superpower for scalability: because no server needs to "remember" you, any server behind a load balancer can handle any request, which is exactly what horizontal scaling needs.

Every HTTP request has a method, a URL, headers, and optionally a body. Every response has a status code, headers, and optionally a body. Let's go through the parts that matter most in system design.

### HTTP Methods

The methods — sometimes called verbs — tell the server what kind of action you want:

- `GET` retrieves a resource. It should be safe (no side effects) and idempotent (calling it many times gives the same result).
- `POST` creates a new resource or triggers an action. Not idempotent — calling it twice usually creates two things.
- `PUT` replaces a resource entirely. It is idempotent — sending the same PUT five times results in the same final state.
- `PATCH` partially updates a resource.
- `DELETE` removes a resource. Idempotent — deleting something that's already gone still results in "it's gone."

That idempotency distinction is one of the most common interview questions, because it directly affects how you handle retries in a distributed system. If a client times out waiting for a response to a PUT and retries, that's safe. Retry a POST blindly, and you might create a duplicate order. This is exactly why real systems add idempotency keys on top of POST when retries are needed — a concept we'll dig into properly later in this course.

### Status Codes

Status codes are grouped into five ranges, and knowing the ranges cold will make you sound sharp in any interview:

- **1xx** — informational, rarely seen directly.
- **2xx** — success. `200 OK`, `201 Created`, `204 No Content`.
- **3xx** — redirection. `301 Moved Permanently`, `304 Not Modified` — that last one is huge for caching, which we'll cover in Module 4.
- **4xx** — client error. `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests`.
- **5xx** — server error. `500 Internal Server Error`, `502 Bad Gateway`, `503 Service Unavailable`.

Notice 401 vs 403 trips a lot of people up: 401 means "I don't know who you are, please authenticate"; 403 means "I know who you are, and you're not allowed." And 429 is one you'll see constantly once we get to rate limiting later in this course.

### Headers, and Then HTTPS

Headers carry metadata: `Content-Type` tells the receiver how to parse the body (JSON, HTML, etc.), `Authorization` carries credentials like a bearer token, `Cache-Control` governs caching behavior, and so on. They're how HTTP stays flexible without changing the core protocol.

Now — HTTPS. HTTPS is just HTTP running over TLS, Transport Layer Security. Plain HTTP sends everything in cleartext: anyone on the network path — a coffee-shop Wi-Fi snooper, an ISP, a malicious router — can read or tamper with it. TLS wraps the connection in encryption using a handshake that does three things: it authenticates the server's identity via a certificate, negotiates a shared symmetric encryption key using asymmetric cryptography, and then encrypts all HTTP traffic in that session with that fast symmetric key. In system design terms, HTTPS is non-negotiable for anything handling real user data, and it has a real performance cost — the TLS handshake adds round trips, which is why techniques like TLS session resumption and HTTP/2's connection multiplexing matter at scale. It's also worth knowing HTTP/1.1 opens one connection per request-ish (with keep-alive helping), while HTTP/2 multiplexes many requests over a single connection, and HTTP/3 moves the transport itself onto QUIC over UDP to avoid head-of-line blocking. You don't need the deep protocol internals for most interviews, but knowing HTTPS = HTTP + TLS, and why it costs a bit of latency, is expected knowledge.

### REST APIs

REST — Representational State Transfer — is an architectural style for designing APIs on top of HTTP, defined by Roy Fielding in his 2000 dissertation. It's not a protocol or a standard; it's a set of constraints that, followed together, produce APIs that are consistent, cacheable, and easy to reason about.

The core idea: everything is a **resource**, identified by a URL, and you act on that resource using standard HTTP methods. So instead of an endpoint like `/getUserById?id=5` or `/createNewOrder`, REST says: the resource is `/users/5`, and the action comes from the HTTP method. `GET /users/5` reads it. `PUT /users/5` replaces it. `DELETE /users/5` removes it. `POST /users` creates a new one under the collection.

Other REST principles worth knowing: it should be stateless (each request self-contained, which aligns with HTTP itself), resources should be cacheable where appropriate, and a well-designed REST API uses proper status codes and nests resources logically — `/users/5/orders` for the orders belonging to user 5.

Good REST design practices: use nouns for resource paths, not verbs; use plural resource names consistently (`/users` not `/user`); use query parameters for filtering, sorting, and pagination — `/users?status=active&page=2`; version your API, typically via the URL path (`/v1/users`) or a header; and return meaningful status codes rather than always returning 200 with an error message buried in the body.

### Real-World Example

Think about a typical e-commerce checkout flow. `GET /products/123` fetches product details — cacheable, safe, idempotent. `POST /cart/items` adds an item to your cart — not idempotent, since calling it twice adds two items. `PUT /cart/items/456` updates the quantity of a specific cart item to an exact value — idempotent. `POST /orders` finally places the order — and because retries of POST are dangerous, mature systems attach an `Idempotency-Key` header so that if the client's network drops and it retries, the server recognizes the duplicate and returns the original order instead of creating a second one. Notice how naturally the resource model — products, cart, cart items, orders — maps onto URLs, and how the HTTP method alone tells you the intent without reading documentation.

### Recap

Let's tie it together. HTTP is a stateless, request-response, application-layer protocol with methods, status codes, and headers. HTTPS wraps it in TLS for encryption and authentication. REST is an architectural style that maps CRUD-like operations onto resources identified by URLs, using HTTP methods to express intent. Get comfortable with idempotency, the status code ranges, and clean resource naming — these come up constantly, both in interviews and in real production API design.

### What's Next

Now that we know how a single client talks to a single server, the natural next question is: what happens when one server isn't enough? In the next video, we're covering load balancing — how traffic gets intelligently spread across many servers, the algorithms behind it, and the crucial difference between Layer 4 and Layer 7 load balancing.

## Key Takeaways

- HTTP is stateless and request-response; statelessness is what makes horizontal scaling behind a load balancer possible.
- Idempotency matters for retries: GET, PUT, DELETE are idempotent; POST and PATCH generally are not.
- Status code ranges: 2xx success, 3xx redirect, 4xx client error, 5xx server error — know 401 vs 403 and 429.
- HTTPS = HTTP + TLS: it authenticates the server, encrypts traffic, and adds handshake latency that connection reuse and HTTP/2+ help offset.
- REST models everything as resources addressed by URLs, using HTTP methods to express the action — not verbs baked into the path.
