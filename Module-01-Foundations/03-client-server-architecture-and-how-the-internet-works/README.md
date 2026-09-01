# Client-Server Architecture & How the Internet Works (DNS, IP, TCP/IP, HTTP)

**Difficulty:** Beginner
**Estimated video length:** 14-18 min
**Prerequisites:** [01 - What is System Design?](../01-what-is-system-design/README.md), [02 - Functional vs Non-Functional Requirements](../02-functional-vs-non-functional-requirements/README.md)

## Learning Objectives

- Explain the client-server model and how it differs from peer-to-peer architecture.
- Describe, step by step, what happens when you type a URL into a browser and hit Enter.
- Understand what DNS is and why it translates domain names into IP addresses.
- Understand what an IP address is and the basic role of the TCP/IP protocol suite.
- Understand what HTTP is, how requests/responses are structured, and where it fits in the stack.

## Script

### Hook / Intro

You type "google.com" into your browser, hit Enter, and a fraction of a second later, a page appears. That single, mundane action actually triggers an incredible chain of events involving multiple protocols, servers scattered across the globe, and several layers of networking — all completing in well under a second. If you're going to design systems that live on the internet, you need to understand this pipeline, because every request your system ever receives travels through it. Let's trace it from click to response.

### The Client-Server Model

First, the foundational architecture pattern: **client-server**. In this model, a **client** — your browser, your phone's app, or another backend service — initiates a request. A **server** — a machine (or fleet of machines) somewhere else — listens for requests, processes them, and sends back a response. The client asks; the server answers. This is asymmetric and it's deliberate: servers are provisioned, maintained, and scaled by whoever runs the service, while clients can be wildly varied devices you don't control.

This is different from **peer-to-peer (P2P)** architecture, where every participant is both a client and a server to every other participant — think BitTorrent, where you download pieces of a file from other users while also uploading pieces to them. Most of the systems we'll design in this course use the client-server model, because it's easier to reason about, secure, and scale in a controlled way.

### Step 1: DNS — Finding the Address

When you type "google.com," your computer doesn't actually know where that is. Computers on the internet find each other using numerical addresses called **IP addresses** (like 142.250.190.14), not human-friendly names. So the very first step is translating "google.com" into an IP address — and that's the job of **DNS**, the Domain Name System.

Think of DNS like a phone book for the internet. Your computer asks a DNS resolver, "what's the IP address for google.com?" That resolver checks a local cache first — maybe your browser or operating system already knows the answer from a recent visit. If not, it asks a chain of DNS servers: a root server, then a server responsible for the ".com" domain, then finally the authoritative server for "google.com," which returns the actual IP address. This whole lookup usually takes milliseconds and is heavily cached at every layer to keep it fast, since doing this full lookup on every single request would be painfully slow.

### Step 2: Establishing a Connection with TCP/IP

Now that your browser has an IP address, it needs to actually establish a reliable connection to that machine. This is where the **TCP/IP** protocol suite comes in.

**IP (Internet Protocol)** handles addressing and routing — it's responsible for getting individual packets of data from your machine to the destination machine, hopping across many intermediate routers along the way. But IP alone doesn't guarantee packets arrive in order, or even arrive at all.

That's where **TCP (Transmission Control Protocol)** comes in. TCP sits on top of IP and adds reliability: it breaks data into packets, numbers them, confirms each one was received (retransmitting any that are lost), and reassembles them in the correct order on the other end. Before any data is sent, TCP performs a **three-way handshake** — the client sends a SYN (synchronize) packet, the server responds with a SYN-ACK, and the client confirms with an ACK. Only after this handshake completes is the connection considered "established," and actual data can start flowing.

This is why TCP is described as "connection-oriented and reliable" — it trades a small amount of upfront latency (the handshake) for a strong guarantee that your data arrives intact and in order. This matters enormously for anything where correctness matters more than raw speed, like loading a webpage or transferring a file.

### Step 3: HTTP — Speaking the Same Language

With a TCP connection established, your browser can now send an actual request for the webpage. This is done using **HTTP (HyperText Transfer Protocol)** — an application-level protocol that defines the *format* of the request and response, riding on top of the TCP connection we just built.

An HTTP request has a **method** (like GET to retrieve something, or POST to submit something), a **path** (like `/search`), **headers** (metadata like what content types the client accepts, or authentication tokens), and optionally a **body** (data being sent, common in POST requests). The server processes this request and sends back an HTTP **response**, which includes a **status code** (like 200 for success, 404 for not found, or 500 for server error), headers, and a body — usually HTML, JSON, or other data.

Modern web traffic typically uses **HTTPS**, which is HTTP layered on top of **TLS (Transport Layer Security)** for encryption — meaning everything sent between client and server is scrambled so eavesdroppers can't read it, and both sides can verify who they're talking to. We'll dig much deeper into HTTP, REST APIs, and status codes in Module 2.

### Putting It All Together

So here's the full sequence, start to finish: you type a URL, your browser (the client) asks DNS to resolve the domain into an IP address, your machine performs a TCP three-way handshake with the server at that IP to establish a reliable connection, layers TLS encryption on top if it's HTTPS, and then sends an HTTP request. The server processes the request — possibly talking to its own databases and caches behind the scenes — and sends back an HTTP response, which your browser renders as the page you see. All of this, across potentially thousands of miles of physical cable and multiple routers, typically completes in well under a second.

### Real-World Example

Consider what happens when you load a news website. DNS resolves the domain to one of possibly many servers behind a load balancer (a concept we'll cover in Module 2). TCP establishes a connection to whichever server the load balancer picks. An HTTPS request goes out, the server fetches the article content — maybe from a cache to avoid hitting the database every time — and returns HTML, CSS, and JavaScript. Your browser then makes many *more* HTTP requests for images, fonts, and scripts referenced in that HTML, each potentially going through this same DNS-to-TCP-to-HTTP pipeline, though usually reusing the already-established connection for efficiency.

### Recap

Let's recap. Client-server architecture separates request-initiators (clients) from request-handlers (servers). Getting a webpage involves three major steps: DNS resolves a domain name into an IP address; TCP/IP establishes a reliable connection between client and server through a three-way handshake; and HTTP defines the structure of the actual request and response exchanged over that connection. Every system you design in this course sits on top of this stack.

### What's Next

Now that we understand how a single request travels from client to server, the natural next question is: what happens when *one* server isn't enough to handle all your traffic? In the next video, we'll cover the two fundamental strategies for scaling a system — vertical scaling and horizontal scaling — and why almost every large system eventually has to choose the horizontal path. See you there.

## Key Takeaways

- Client-server architecture: clients initiate requests, servers process them and respond; contrasts with peer-to-peer.
- DNS translates human-readable domain names into numerical IP addresses, similar to a phone book.
- TCP/IP provides reliable, ordered delivery of data between machines via a three-way handshake (SYN, SYN-ACK, ACK), with IP handling addressing/routing and TCP handling reliability.
- HTTP is an application-level protocol defining the format of requests (method, path, headers, body) and responses (status code, headers, body); HTTPS adds TLS encryption on top.
- The full request path (DNS → TCP handshake → HTTP request/response) underlies every system you'll design in this course.
