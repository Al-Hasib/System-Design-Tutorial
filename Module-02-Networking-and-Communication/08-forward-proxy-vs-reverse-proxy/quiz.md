# Practice & Interview Questions

**1. In one sentence, what's the difference between a forward proxy and a reverse proxy?**
A forward proxy sits in front of clients and hides/represents them from the server; a reverse proxy sits in front of servers and hides/represents them from the client.

**2. Whose identity does the destination server see in a forward proxy setup?**
The destination server only sees the forward proxy's IP/identity — it has no visibility into the actual originating client.

**3. Give two real-world use cases for a forward proxy.**
Corporate content filtering/monitoring of employee outbound traffic, and anonymizing/masking a client's identity or location (e.g., bypassing geo-restrictions or hiding internal client IPs).

**4. Give two real-world use cases for a reverse proxy.**
TLS/SSL termination on behalf of backend servers, and caching or load balancing requests across a pool of backend application servers.

**5. Is a load balancer a separate concept from a reverse proxy, or a special case of one?**
An L7 load balancer is generally a special case of a reverse proxy — a reverse proxy that includes load-distribution algorithms as one of its features. Reverse proxy is the broader category; load balancing is one common function among others like TLS termination and caching.

**6. Why does a reverse proxy improve security for backend servers?**
It hides the internal network topology and real IP addresses of backend servers from external clients, so attackers can't directly target them; it also centralizes where security controls (TLS, WAF rules, rate limiting) are applied.

**7. A company wants to prevent employees from accessing certain websites during work hours and log all outbound traffic for compliance. Which type of proxy do they need, and why?**
A forward proxy — it's deployed on the client (employee) side, can inspect/filter/block outbound requests before they leave the network, and log all traffic since every employee's requests pass through it.

**8. Why does the client in a reverse proxy setup typically have no idea how many backend servers exist?**
Because the reverse proxy presents a single unified address/identity to the client and internally forwards the request to whichever backend server it chooses — the backend topology is completely abstracted away from the client's perspective.

**9. Can a single infrastructure use both a forward proxy and a reverse proxy at the same time? Give an example.**
Yes — a company can run a forward proxy for its employees' outbound internet traffic (protecting/monitoring clients) while separately running a reverse proxy (like NGINX) in front of its public-facing web servers (protecting/optimizing the server side). These serve entirely different traffic flows and purposes.

**10. Why might a reverse proxy cache responses, and what kind of content is best suited for this?**
Caching at the reverse proxy avoids repeatedly hitting backend servers for identical requests, reducing load and latency. It's best suited for static or infrequently-changing content (images, CSS/JS, product pages that don't change per-request) rather than highly personalized or rapidly changing data.

**11. What's a potential downside of routing all outbound traffic through a single forward proxy?**
It becomes a single point of failure and a potential bottleneck for all outbound client traffic — if it goes down or gets overloaded, no client behind it can reach the internet; it also needs to be scaled/made redundant like any critical infrastructure component.

**12. How would you explain to an interviewer why "reverse proxy" and "API Gateway" (covered in the next video) are related but not identical?**
An API Gateway builds on reverse-proxy concepts — it also sits in front of backend services and hides their topology — but adds application-aware capabilities specific to APIs and microservices, like authentication/authorization, rate limiting, request/response transformation, and aggregating calls to multiple services, going beyond what a plain reverse proxy typically does.
