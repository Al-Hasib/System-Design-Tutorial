# CDN (Content Delivery Network) Explained

**Difficulty:** Intermediate
**Estimated Length:** 10-13 min
**Prerequisites:** [17 - Caching Strategies & Cache Invalidation](../17-caching-strategies-and-cache-invalidation/README.md), [03 - Client-Server Architecture and How the Internet Works](../../Module-01-Foundations/03-client-server-architecture-and-how-the-internet-works/README.md)

## Learning Objectives

- Explain what a CDN is and the core problem it solves: network latency caused by physical distance
- Describe how edge servers, PoPs (Points of Presence), and origin servers relate to each other
- Differentiate between static and dynamic content caching at the edge
- Understand CDN request routing techniques (Anycast, DNS-based routing)
- Recognize CDN cache invalidation/purging and typical use cases (static assets, video streaming, DDoS protection)

## Script

### Hook / Intro

Have you ever noticed that a website loads almost instantly no matter where in the world you are, even though the company running it might have all its servers in one country? That's not magic — it's a Content Delivery Network, or CDN, quietly doing its job in the background. In the last video, we talked about caching data close to your application. Today we're zooming out to a global scale: caching content close to your *users*, wherever they happen to be on the planet. By the end of this video, you'll understand how CDNs like Cloudflare, Akamai, and Amazon CloudFront work, and why almost every serious production system uses one.

### The Problem: Physical Distance is Latency

Let's start with the core problem. The internet is fast, but it's not instant — data still has to physically travel, and it's bound by the speed of light. If your origin server is in Virginia, USA, and a user is browsing from Singapore, that round trip is roughly 15,000 kilometers each way. Even at the speed of light in fiber optic cable, that's around 100 milliseconds just for the network round trip, before the server has even done any work. Now multiply that by every image, script, and stylesheet a webpage needs to load, and suddenly a page that feels instant in Virginia feels sluggish in Singapore.

A CDN solves this by placing copies of your content on servers physically distributed around the world, so users fetch content from a nearby server instead of from your single, distant origin.

### Edge Servers and Points of Presence (PoPs)

CDN providers operate thousands of servers positioned at what are called **Points of Presence**, or PoPs — data centers spread across cities and countries worldwide. The servers inside these PoPs are often called **edge servers**, because they sit at the "edge" of the network, as close to end users as possible, as opposed to your **origin server**, which is where the actual, authoritative copy of your content lives — your own infrastructure, or a cloud provider's servers.

When a user requests content, instead of that request traveling all the way to your origin, it's routed to the nearest edge server. If that edge server already has a cached copy of the content — a cache hit — it serves it directly, often in a few milliseconds. If it doesn't have it yet — a cache miss — the edge server fetches it from the origin once, caches it locally, and serves it to the user. Every subsequent nearby user then benefits from that cached copy, without the origin being touched again until the cache expires or is purged.

### Static vs Dynamic Content

CDNs are historically best known for caching **static content** — things that don't change per-user or per-request: images, CSS and JavaScript files, videos, downloadable files, and whole HTML pages that are the same for everyone. This is the classic, easy case: cache it at the edge with a TTL, and you're done.

**Dynamic content** — personalized pages, API responses, anything that depends on the specific user or changes frequently — used to be considered "impossible" to cache at a CDN. But modern CDNs have gotten much smarter. Techniques like edge computing (running small pieces of application logic directly on edge servers), micro-caching (caching dynamic responses for just a few seconds), and caching based on request headers or cookies now let CDNs accelerate even semi-dynamic content. Still, the fundamental rule holds: the more universally shareable a response is across users, the easier and safer it is to cache at the edge.

### Request Routing: How Do You Find "the Nearest Server"?

So how does a user's request actually find the nearest edge server? There are two main techniques. The first is **DNS-based routing**: when your browser looks up the CDN's domain, the CDN's DNS system returns a different IP address depending on where in the world your DNS query came from, effectively directing you to a nearby PoP. The second, more modern technique is **Anycast routing**: the exact same IP address is announced from many different physical locations simultaneously, and the internet's own routing protocols (BGP) automatically send your traffic to whichever announcing location is "closest" in terms of network hops — no per-user DNS trickery required. Anycast is how large providers like Cloudflare achieve extremely fast, resilient global routing.

### CDN Cache Invalidation (Purging)

Just like the application caches from our last video, CDN caches need invalidation too. Most CDNs support **TTL-based expiration**, configured via HTTP cache-control headers on your origin's responses, telling every edge server how long to keep a copy before re-checking the origin. For urgent changes, CDNs also offer **manual purging** — an API call or dashboard action that says "throw away your cached copy of this file right now, everywhere," useful when you deploy a new version of a JavaScript bundle or need to correct a mistakenly published image immediately.

### Real-World Example

Think about Netflix or YouTube streaming a popular video. If every single viewer's video stream had to come directly from a single origin data center, that data center — and the network links leading to it — would be instantly overwhelmed, and users far from that data center would see constant buffering. Instead, video content is pushed out to CDN edge servers in advance, or cached after the first request in each region, so a viewer in Australia streams from an edge server in Australia, not from a data center in the US. This is also why CDNs are a critical part of surviving traffic spikes — a viral article or product launch can be served almost entirely from cached edge copies, with the origin barely noticing the load. As a bonus, because CDNs sit in front of your origin and absorb massive amounts of traffic, they're also commonly used as a first line of defense against DDoS attacks, since malicious traffic gets absorbed and filtered at the edge before it ever reaches your real servers.

### Recap

Let's recap. A CDN is a globally distributed network of edge servers that cache your content physically close to your users, dramatically cutting latency versus hitting a single distant origin. Static content is the classic use case, but modern CDNs increasingly handle dynamic content too, through edge computing and micro-caching. Requests are routed to the nearest edge server via DNS-based routing or Anycast. And just like any cache, CDN content needs invalidation — through TTLs and cache-control headers, or manual purges when you need changes to take effect immediately.

### What's Next

We've now covered caching inside your application (cache-aside, write-through, etc.) and caching at the network edge with CDNs. But what about caching that's shared across many application servers, in a dedicated, purpose-built layer? In the next video, we'll dig into distributed caching with Redis and Memcached — two of the most widely used caching technologies in the industry — and compare how they work, when to use each, and how they fit into the architectures we've already discussed.

## Key Takeaways

- CDNs solve the physical-distance latency problem by caching content on edge servers close to end users, instead of forcing every request to reach a single distant origin.
- PoPs (Points of Presence) host edge servers around the world; the origin server holds the authoritative copy of content.
- Static content (images, JS/CSS, video, static HTML) is the classic CDN use case; dynamic content can be accelerated via edge computing and micro-caching.
- CDNs route users to the nearest edge server via DNS-based routing or Anycast (BGP-based) routing.
- CDN caches are invalidated via TTLs/cache-control headers or manual purge APIs.
- Beyond performance, CDNs also help absorb traffic spikes and provide a first line of defense against DDoS attacks.
