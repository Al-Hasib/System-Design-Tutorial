# Load Balancing Explained (Algorithms & L4 vs L7)

**Difficulty:** Intermediate
**Estimated length:** 12-15 min
**Prerequisites:** [06 - HTTP/HTTPS & REST APIs Explained](../06-http-https-and-rest-apis/README.md), [04 - Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md)

## Learning Objectives

- Explain why load balancers are essential once a system scales horizontally.
- Compare the common load balancing algorithms: round robin, least connections, weighted variants, IP hash, and consistent hashing.
- Distinguish Layer 4 (transport) from Layer 7 (application) load balancing and know when to use each.
- Describe how health checks keep a load balancer's server pool accurate.
- Identify single points of failure in a load balancing setup and how to remove them.

## Script

### Hook / Intro

In the last video we established that HTTP is stateless — which is great, because it means any server can technically answer any request. But that raises an obvious question: if I have ten identical servers capable of handling a request, which one actually gets it? That decision — made on every single request, thousands or millions of times a second at scale — is the job of a load balancer. Today we're breaking down exactly how load balancers decide, the major algorithms they use, and the fundamental difference between balancing at Layer 4 versus Layer 7. This is one of the most consistently tested topics in system design interviews, so let's get it rock solid.

### Why We Need Load Balancing

Picture a single server handling all your traffic. It has a hardware ceiling — a maximum number of requests per second before CPU, memory, or network bandwidth becomes the bottleneck. Vertical scaling — buying a bigger machine — helps for a while, but it's expensive and has a hard limit. So we scale horizontally: run the same application on many servers. But now clients need a single, stable address to talk to, and the requests need to be spread across the fleet intelligently. That's the load balancer: it sits in front of your server pool, receives all incoming traffic, and decides which backend server handles each request.

Beyond just spreading load, a good load balancer gives you two other huge wins. First, availability: if one server crashes, the load balancer detects it via health checks and simply stops sending it traffic — the failure becomes invisible to users. Second, it enables zero-downtime deployments: you can take servers out of rotation, update them, and add them back, all without users noticing.

### Load Balancing Algorithms

Let's go through how a load balancer actually picks a server. This is more nuanced than "just pick one at random."

**Round robin** is the simplest: requests are handed to servers in strict rotation — server 1, then 2, then 3, then back to 1. It's simple and fair when all servers are identical and all requests are roughly equal cost. **Weighted round robin** extends this by giving more powerful servers a higher weight, so a beefier server gets proportionally more requests.

**Least connections** routes each new request to whichever server currently has the fewest active connections. This is smarter than round robin when requests vary wildly in cost — a server stuck processing a slow request won't keep getting piled on. **Weighted least connections** again factors in server capacity.

**IP hash** computes a hash of the client's IP address and uses it to consistently map that client to the same backend server. This is useful when you need "sticky sessions" — for example, if a server is holding some in-memory session state for that user. The tradeoff is that if a server goes down, everyone hashed to it needs to be redistributed, and hash-based approaches can cause uneven load if client IP distribution is skewed.

There's also **least response time**, which factors in both active connections and observed latency, and — worth a callout since we'll cover it in-depth later in the course — **consistent hashing**, which minimizes the redistribution problem when servers are added or removed, commonly used for cache and shard routing rather than plain web traffic.

### Layer 4 vs Layer 7 Load Balancing

This is the distinction interviewers love to probe, so let's be precise. It refers to which layer of the OSI networking model the load balancer operates at.

A **Layer 4 (L4) load balancer** works at the transport layer — it looks at TCP/UDP information: source and destination IP addresses and ports. It makes routing decisions without ever looking inside the packet's actual content or understanding HTTP at all. It just sees "connection from IP A to IP B" and forwards packets. This makes L4 balancing extremely fast and low-overhead — it can push huge throughput with minimal CPU cost — but it's also "dumb" in the sense that it can't make decisions based on the request's content, like the URL path or headers.

A **Layer 7 (L7) load balancer** works at the application layer — it actually understands HTTP. It can read the URL path, headers, cookies, even the request body, and route based on that. This means an L7 balancer can send `/api/images/*` to one service and `/api/checkout/*` to another, terminate TLS on behalf of the backend servers, inject or rewrite headers, and implement sticky sessions via cookies rather than just IP hashing. The cost is more CPU overhead per request, since it has to actually parse and understand each HTTP request rather than just shuffling packets.

A simple way to remember it: L4 routes connections, L7 routes requests. In practice, most modern production systems use L7 load balancers — like NGINX, HAProxy in L7 mode, AWS Application Load Balancer, or Envoy — because the intelligent routing capabilities are worth the extra CPU cost, and hardware has made that cost fairly negligible at most scales. L4 balancers — like AWS Network Load Balancer — still shine for extremely high-throughput, low-latency scenarios, or for non-HTTP protocols entirely.

### Health Checks and Avoiding a Single Point of Failure

A load balancer is only as good as its knowledge of which servers are actually healthy. That's done through health checks — the load balancer periodically pings each backend, often via a lightweight `/health` HTTP endpoint, and if a server fails to respond correctly a set number of times, it's pulled from the rotation automatically. This is what makes failure invisible to the end user.

But here's a subtlety that trips people up in interviews: the load balancer itself is a server, and a single load balancer is a single point of failure. If it goes down, your whole fleet of perfectly healthy backend servers becomes unreachable. Production systems solve this with redundancy — running multiple load balancer instances behind a mechanism like DNS round robin, a floating/virtual IP with a protocol like VRRP, or a cloud provider's managed load balancer service that's inherently redundant across availability zones. The general principle: never let load balancing itself become the bottleneck or the single point of failure it was meant to eliminate.

### Real-World Example

Think about a video streaming platform's API tier during a big live event. Millions of clients connect simultaneously. An L7 load balancer sits at the edge, terminating TLS so backend servers don't each need to manage certificates, and it routes based on path: requests to `/video/manifest` go to a lightly-loaded manifest service, `/chat/messages` goes to a separate chat service, and health checks continuously monitor both pools. Within the video manifest pool, it might use least-connections, since manifest requests can vary in cost depending on video quality tiers. If a manifest server starts throwing errors under load, health checks catch it within seconds and traffic reroutes automatically — users just see a slightly higher, but non-fatal, average latency instead of failed requests.

### Recap

Load balancers exist because horizontal scaling only works if traffic is intelligently spread across many identical servers. The algorithm — round robin, least connections, IP hash, and their weighted variants — determines how that spreading happens. The L4 vs L7 distinction is about what information the load balancer uses to decide: L4 sees only IP/port and is extremely fast; L7 understands HTTP itself and can make much smarter routing decisions at a slightly higher CPU cost. Health checks keep the pool accurate, and redundant load balancers prevent the balancer itself from becoming a single point of failure.

### What's Next

Load balancers are one flavor of a broader concept: something sitting between the client and the real servers. In the next video, we'll zoom out and cover proxies in general — specifically the difference between a forward proxy, which represents and protects clients, and a reverse proxy, which represents and protects servers. You'll see how a reverse proxy and a load balancer often overlap in practice.

## Key Takeaways

- Load balancers distribute traffic across a server pool, enabling horizontal scaling, high availability, and zero-downtime deployments.
- Common algorithms: round robin, weighted round robin, least connections, IP hash — each suited to different traffic patterns.
- L4 load balancing routes based on IP/port (transport layer) — fast but content-blind. L7 load balancing understands HTTP and routes based on paths, headers, cookies — smarter but costlier.
- Health checks let a load balancer automatically remove unhealthy servers from rotation, making failures invisible to users.
- A single load balancer is a single point of failure — production systems run redundant load balancers to avoid this.
