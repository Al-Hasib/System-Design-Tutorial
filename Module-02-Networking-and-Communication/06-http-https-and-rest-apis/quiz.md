# Practice & Interview Questions

**1. What does it mean for HTTP to be "stateless," and why is that property important for scalability?**
Each request is handled independently, with no memory of prior requests carried on the server. This means any server behind a load balancer can handle any client's request, which is what makes horizontal scaling of stateless web servers straightforward.

**2. Explain the difference between PUT and PATCH.**
PUT replaces the entire resource with the payload provided (idempotent). PATCH applies a partial update, modifying only the specified fields, and is generally not guaranteed to be idempotent.

**3. Why is idempotency important when designing APIs that might be retried by a client?**
If a network call times out, the client may not know whether the server processed it, so it often retries. Idempotent operations (GET, PUT, DELETE) are safe to retry blindly. Non-idempotent operations like POST can create duplicates on retry unless protected by something like an idempotency key.

**4. What's the difference between a 401 and a 403 status code?**
401 Unauthorized means the server doesn't know who the caller is — authentication is missing or invalid. 403 Forbidden means the server knows who the caller is but they don't have permission to perform the action.

**5. What does HTTPS add on top of HTTP, and what is the performance cost?**
HTTPS wraps HTTP in TLS, which authenticates the server via a certificate and encrypts all traffic. The cost is the TLS handshake, which adds extra round trips before the first request can be sent — mitigated by session resumption, TLS 1.3's 1-RTT/0-RTT modes, and connection reuse.

**6. A client calls `POST /orders` and the connection drops before it receives a response. Design a way to make this retry-safe.**
Have the client generate a unique idempotency key (e.g., a UUID) and send it in an `Idempotency-Key` header. The server stores completed request results keyed by that value; if it sees the same key again, it returns the original response instead of creating a second order.

**7. What are the core constraints of REST as an architectural style?**
Resources identified by URLs, manipulation via a uniform set of methods (the standard HTTP verbs), statelessness, cacheability where appropriate, and a consistent/uniform interface across the API.

**8. Design REST endpoints for a blogging platform where users write posts and posts have comments.**
`GET /users/{id}/posts` (list a user's posts), `POST /posts` (create a post), `GET /posts/{id}` (read one), `PUT /posts/{id}` (replace), `DELETE /posts/{id}` (remove), `GET /posts/{id}/comments` (list comments), `POST /posts/{id}/comments` (add a comment) — resources are nouns, nesting expresses the comment-belongs-to-post relationship.

**9. Why might returning `200 OK` with an error message in the JSON body be considered bad REST design?**
It breaks the uniform interface — clients (and infrastructure like caches, monitoring, and load balancers) rely on status codes to determine success/failure without parsing the body. Returning 200 for errors forces every caller to inspect the payload, defeating standard HTTP semantics and tooling.

**10. What is the difference between HTTP/1.1 and HTTP/2 that matters for performance?**
HTTP/1.1 effectively needs multiple connections (or serialized requests with keep-alive) to fetch resources in parallel, leading to overhead. HTTP/2 multiplexes many requests and responses over a single TCP connection and compresses headers, reducing latency and connection overhead, particularly for pages/APIs with many concurrent calls.

**11. Why is GET expected to have no side effects (be "safe")?**
Because browsers, proxies, and crawlers may issue GET requests automatically (prefetching, retries, caching) without the user's explicit intent for each one. If GET triggered state changes, those automatic requests could cause unintended actions.

**12. In a system design interview, how would you justify choosing REST over a custom RPC-style API for a public-facing API?**
REST's resource-oriented design leverages well-understood HTTP semantics (methods, status codes, caching), making it easier for third-party developers to learn, easier to cache at the HTTP layer (CDNs, browsers), and easier to evolve/version predictably compared to a bespoke RPC protocol.
