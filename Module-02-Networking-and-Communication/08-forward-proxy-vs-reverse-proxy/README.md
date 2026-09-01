# Forward Proxy vs Reverse Proxy

**Difficulty:** Intermediate
**Estimated length:** 10-12 min
**Prerequisites:** [07 - Load Balancing Explained](../07-load-balancing-explained/README.md), [06 - HTTP/HTTPS & REST APIs Explained](../06-http-https-and-rest-apis/README.md)

## Learning Objectives

- Define what a proxy is and why an intermediary between client and server is useful.
- Explain how a forward proxy works and who it serves.
- Explain how a reverse proxy works and who it serves.
- Compare forward and reverse proxies side by side, including real products that implement each.
- Understand how a reverse proxy relates to, and differs from, a load balancer.

## Script

### Hook / Intro

In the last video, we talked about load balancers sitting in front of a server pool, deciding which backend handles each request. That's actually one specific job performed by a broader category of component: the proxy. Anytime something sits between a client and a server and relays traffic on someone's behalf, that's a proxy. But here's the twist that confuses a lot of people early on — there are two fundamentally different kinds of proxies, and they exist to protect completely different parties. A forward proxy works for the client. A reverse proxy works for the server. Get that one sentence locked in, and everything else in this video will click into place.

### What a Proxy Is, Generally

A proxy is an intermediary that sits between a client and a server and forwards requests and responses on their behalf. Instead of the client talking directly to the destination server, it talks to the proxy, and the proxy talks to the destination — or vice versa. Because the proxy sits in the middle of that conversation, it can do a lot more than just forward bytes: it can cache responses, filter or block requests, log traffic, rewrite content, add security, and hide the identity of whichever side it represents.

The key question that tells you which kind of proxy you're dealing with is simple: **whose identity is it hiding, and whose interests is it protecting?**

### Forward Proxy

A forward proxy sits in front of a group of clients, and from the server's point of view, it looks like the traffic is coming from the proxy, not the actual client. The client explicitly configures itself to send its traffic through the forward proxy.

Think about a corporate office network. Every employee's laptop is configured to route all outbound web traffic through a company proxy server. When someone visits a website, the destination server only ever sees the proxy's IP address — it has no idea which specific employee made the request. The company uses this for a bunch of reasons: content filtering (blocking access to certain categories of sites), monitoring and logging employee traffic for compliance, caching frequently accessed resources so multiple employees don't redundantly re-download the same content, and masking internal IP addresses and network topology from the outside world for security.

Another everyday example is a VPN or a service like a public web proxy used to bypass geographic restrictions — your traffic goes out through the proxy's location, so the destination server thinks the request is coming from wherever the proxy is, not from you. The defining trait: **a forward proxy protects and represents the client**, hiding the client from the server.

### Reverse Proxy

A reverse proxy does the mirror-image job: it sits in front of a group of servers, and from the client's point of view, it looks like the reverse proxy *is* the server. The client has no idea, and doesn't need to know, that there's a whole fleet of actual backend servers behind it.

This is the pattern almost every production web architecture uses. When you hit a website, you're very often not talking directly to the application server — you're talking to a reverse proxy like NGINX, HAProxy, or a cloud load balancer, which then forwards your request internally to one of many backend servers. Reverse proxies exist for the server side's benefit: they terminate TLS so backend servers don't each need certificate management, they can cache static or frequently requested content close to the edge, they compress responses, they hide the internal network topology and real server IPs from the outside world (a real security benefit — attackers can't directly target backend servers they can't even see), and — this is the overlap point — they very often perform load balancing across the backend pool as one of their jobs.

That last point is important: **a load balancer, when it operates at the application layer in front of your servers, is typically implemented as a specific kind of reverse proxy.** NGINX and HAProxy are, technically, reverse proxies that happen to include load balancing algorithms as a feature. So the relationship isn't "load balancer vs. reverse proxy" as competing concepts — a reverse proxy is the broader category, and load balancing is one very common function it performs, alongside TLS termination, caching, and request routing.

### Forward vs Reverse: The Clean Comparison

Let's nail the distinction with a clean mental model. In a forward proxy setup, the client knows about and configures the proxy; the server has no idea a proxy is involved at all — it just sees requests coming from what looks like a single client (the proxy's IP). In a reverse proxy setup, it's the opposite: the server side deploys and controls the proxy; the client has no idea there's a whole backend fleet behind it — it just sees what looks like a single server.

Said differently: forward proxy = hides the client from the server. Reverse proxy = hides the server from the client. Both provide anonymity, caching, and security benefits — but for opposite parties.

### Real-World Example

Consider a large e-commerce company. Internally, their corporate employees browse the internet through a forward proxy that filters malicious sites and logs traffic for security compliance — that proxy exists purely to protect and control the company's own clients (its employees). Completely separately, when a customer visits the e-commerce website, their request hits an NGINX reverse proxy at the edge of the company's infrastructure. That reverse proxy terminates TLS, checks a cache for the product page (serving it instantly if cached), and if not cached, load-balances the request across a pool of application servers. The customer's browser never learns there are dozens of backend servers — it only ever "sees" the reverse proxy. Same broad concept — an intermediary relaying traffic — completely different party being served.

### Recap

A proxy is any intermediary between client and server. A forward proxy sits in front of clients and represents them — the server sees the proxy, not the real client — commonly used for content filtering, anonymity, and caching on the client's behalf. A reverse proxy sits in front of servers and represents them — the client sees the proxy, not the real backend fleet — commonly used for TLS termination, caching, hiding internal topology, and load balancing. Remember: a load balancer is very often just a reverse proxy with load-distribution logic built in, not a separate category of thing.

### What's Next

Now that we understand reverse proxies as the front door to a server fleet, the next video takes that idea further into the microservices world: the API Gateway. We'll look at how a gateway builds on reverse-proxy concepts to add authentication, rate limiting, and unified routing across dozens of backend services — and we'll introduce the Backend-for-Frontend pattern for when different client types need different API shapes.

## Key Takeaways

- A proxy is any intermediary that relays traffic between a client and a server on someone's behalf.
- A forward proxy represents the client — the destination server only sees the proxy, not the real client. Used for content filtering, anonymity, caching, and monitoring client traffic.
- A reverse proxy represents the server — the client only sees the proxy, not the real backend fleet. Used for TLS termination, caching, hiding internal topology, and load balancing.
- Load balancing is commonly just one function of a reverse proxy, not a separate architectural category.
- The one-line test: forward proxy hides the client from the server; reverse proxy hides the server from the client.
