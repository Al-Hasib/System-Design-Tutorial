# Web Server Internals: Concurrency, Threading & Content Serving

**Difficulty:** Intermediate
**Estimated length:** 14-18 min
**Prerequisites:** [07 - Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md), [08 - Forward Proxy vs Reverse Proxy](../../Module-02-Networking-and-Communication/08-forward-proxy-vs-reverse-proxy/README.md)

## Learning Objectives

- Explain what a web server actually does with an incoming connection, request, and response.
- Compare the three main concurrency models: process-per-request, thread-per-request, and the event loop.
- Understand the C10K problem and why it forced a shift away from one-thread-per-connection designs.
- Distinguish static content serving from dynamic content serving and explain why each is handled differently.
- Explain how connection keep-alive, thread/worker pools, and reverse-proxy caching interact to determine real-world throughput.

## Script

### Hook / Intro

We've talked about load balancers spreading traffic across servers, and reverse proxies sitting in front of them — but what actually happens *inside* one of those servers when ten thousand requests land on it at the same time? This is the part most backend engineers can use a framework for years without ever having to think about — until the day their app falls over under load and the postmortem says "thread pool exhaustion" or "the event loop got blocked." Today we open up the web server itself: how it handles concurrency, why some frameworks handle 50,000 connections on one box while others tip over at 500, and why serving a static image and serving a database-backed API response are fundamentally different jobs.

### What a Web Server Actually Does

Strip away the framework-specific magic, and a web server does four things, in a loop, for every request: accept a connection, read and parse the request, do whatever work is needed to produce a response (which might be nothing more than reading a file, or might involve hitting a database and three downstream services), and write the response back. The entire question of "web server performance" comes down to how efficiently a server can do step three — the actual work — for many requests *at the same time*, without one slow request stalling all the others.

### Concurrency Models

There have historically been three broad approaches to handling many requests concurrently.

**Process-per-request** — the original CGI model — forks a brand-new operating system process for every incoming request. It's simple and extremely isolated (one crashing request can't take down another), but processes are heavyweight: creating one, giving it its own memory space, and tearing it down again is slow and doesn't scale past a few hundred concurrent requests per machine.

**Thread-per-request** is the classic model used by servers like Apache's `mpm_prefork`/`worker` and many traditional Java application servers: each incoming connection gets its own OS or lightweight thread, which blocks on I/O (like waiting for a database query) until it has a response to send. Threads are cheaper than processes, but each one still costs real memory (often a megabyte or more of stack space) and the OS has to context-switch between them, which has real overhead once you have thousands of threads. This is where the famous **C10K problem** comes from — a 1999 essay pointing out that handling ten thousand concurrent connections with one-thread-per-connection architectures was hitting a wall, because the OS simply couldn't cheaply schedule that many threads.

**Event loop / asynchronous, non-blocking I/O** is the modern answer, used by Node.js, Nginx, and async frameworks in Python (asyncio), Java (Netty), and elsewhere. Instead of one thread per connection, a small, fixed pool of threads (sometimes just one) handles many connections by never blocking: when a request needs to wait on I/O — a database call, a file read, a call to another service — the event loop registers a callback and moves on to service other connections, only coming back to this one when the I/O completes. This means thousands of "waiting" connections cost almost nothing, because none of them are tying up a dedicated thread while they wait. The trade-off: a single long-running, CPU-bound piece of work (like a heavy synchronous computation) blocks the *entire* event loop for every connection it's serving — which is exactly the "I accidentally blocked the event loop" bug that takes down a Node.js server under load. In practice, most production systems today are hybrids: an async event loop handles I/O-bound work, backed by a worker/thread pool for anything genuinely CPU-bound.

### Static vs. Dynamic Content

Not all requests are equal work. **Static content** — an image, a CSS file, a pre-built JavaScript bundle — never changes per-request; the server's job is just "read these bytes from disk (or memory) and send them," which is why static assets are aggressively cached (recall CDNs from Module 4) and why dedicated static file servers (or a CDN edge) can serve them far faster and cheaper than routing them through your application code. **Dynamic content** requires the server to actually do work specific to this request — query a database, apply business logic, personalize a response — before it has anything to send back. This is precisely why production architectures put a reverse proxy or CDN in front of the application server: static assets and cacheable responses get served (or cached) right at the edge, and only requests that truly need dynamic computation ever reach your application's concurrency model at all. The fewer requests that reach your app servers, the further your concurrency model's limits stretch.

### Real-World Example

Imagine an e-commerce product page. The product images and the page's CSS/JS bundle are static — served straight from a CDN edge cache, never touching your application servers. The `GET /products/123` API call that returns current price and stock is dynamic — it has to hit a database (or a cache-aside layer, per Module 4) and can't be served from a CDN edge unless you're comfortable with slightly stale prices. If this API is built on Node.js's event loop, ten thousand concurrent `GET /products/123` calls are cheap to hold "in flight" while waiting on the database — the event loop isn't blocked by any one of them. But if a request needs to generate a PDF invoice synchronously (a genuinely CPU-heavy task), doing that directly on the event loop thread would stall every other request on the server until it finishes — which is exactly the kind of work you'd offload to a background worker pool or a separate service instead.

### Recap

A web server's core loop is accept, read, process, write — and the entire performance story is about how it handles "process" concurrently. Process-per-request and thread-per-request are simple but don't scale past a few thousand connections because of OS-level overhead — the C10K problem. Event-loop, non-blocking I/O scales to tens of thousands of connections by never dedicating a thread to a connection that's just waiting, but is vulnerable to any single piece of CPU-bound work blocking everyone else. And static content should never reach this concurrency model at all — it belongs at a CDN or reverse-proxy edge, leaving your application server's limited concurrency for the dynamic work that actually needs it.

### What's Next

We've now covered how bytes move (TCP/UDP/gRPC) and how a server handles many requests at once. The next question is: once two services decide to exchange data, what format is that data actually in? Next video, we compare JSON, XML, and Protocol Buffers on the things that actually matter — bandwidth, parsing speed, and how gracefully each handles a schema changing over time.

## Key Takeaways

- Every web server does the same four steps per request — accept, read, process, write — and performance is entirely about how "process" scales concurrently.
- Process-per-request and thread-per-request models don't scale past a few thousand connections due to OS memory and context-switching overhead — the C10K problem.
- Event-loop/async, non-blocking I/O (Nginx, Node.js, asyncio) scales to tens of thousands of connections by never tying up a thread on I/O waits, but a blocking CPU-bound task on the loop stalls every connection it serves.
- Static content (images, CSS, prebuilt bundles) should be served from a CDN/reverse-proxy edge, never routed through application concurrency logic.
- Most production systems are hybrids: an async event loop for I/O-bound work, backed by a worker/thread pool for genuinely CPU-bound tasks.
