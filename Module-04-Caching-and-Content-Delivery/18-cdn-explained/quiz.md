# Practice & Interview Questions

**1. What core problem does a CDN solve, and why can't you fix it just by adding more servers at your origin data center?**
A CDN solves the latency caused by physical distance between users and a single origin server — data is bound by the speed of light, so distant users always experience network round-trip delay. Adding more servers at one origin location doesn't reduce that physical distance; you need servers physically closer to users, which is exactly what a CDN's distributed edge network provides.

**2. Define "origin server," "edge server," and "PoP," and explain how they relate.**
The origin server holds the authoritative, canonical copy of the content. A PoP (Point of Presence) is a physical data center location operated by the CDN provider. An edge server is a server inside a PoP that caches copies of origin content and serves it to nearby users, only contacting the origin on a cache miss.

**3. Why is static content easier to cache at the edge than dynamic content?**
Static content (images, CSS, JS, videos) is identical for every user, so a single cached copy can serve everyone in a region. Dynamic content often varies per user or per request (personalization, live data), making a single cached copy unsafe or incorrect to serve broadly unless additional techniques (micro-caching, cache keys per user segment, edge computing) are used.

**4. Explain the difference between DNS-based routing and Anycast routing for directing users to the nearest edge server.**
DNS-based routing resolves the CDN's domain to different IP addresses depending on where the DNS query originates, directing the user toward a nearby PoP. Anycast announces the same IP address from many physical locations simultaneously, letting the internet's own BGP routing automatically send each user's traffic to the topologically nearest announcing location, without relying on DNS resolution behavior.

**5. How does a CDN handle a cache miss at the edge?**
When a requested object isn't cached at the nearest edge server, the edge server fetches it from the origin server, caches a copy locally for future requests, and returns the content to the requesting user. Subsequent nearby requests for the same object are then served directly from the edge cache.

**6. Your company just deployed a critical bug fix to a JavaScript file, but users are still seeing the old buggy version. What are two ways to fix this via the CDN?**
You can issue a manual purge/invalidation request through the CDN's API or dashboard to immediately evict the cached copy across all edge servers. Alternatively, use a cache-busting strategy — deploy the new file under a new versioned URL/filename (e.g., with a content hash) so it's treated as a brand-new, uncached resource rather than relying on the old cached one to expire.

**7. What is micro-caching, and why is it used for semi-dynamic content?**
Micro-caching caches a dynamic response at the edge for a very short duration (e.g., a few seconds) instead of not caching it at all. It's useful for content that changes frequently but where serving a few-seconds-old version to many users is acceptable, letting the CDN absorb huge traffic spikes on otherwise "dynamic" endpoints.

**8. Besides performance, what other benefit do CDNs provide to a production system?**
CDNs act as a buffer in front of the origin, absorbing traffic spikes (e.g., viral content) and filtering malicious traffic, which makes them a common first line of defense against DDoS attacks — the origin server is shielded from most of the volume before it ever arrives.

**9. What HTTP mechanism tells a CDN how long it may cache a piece of content?**
The `Cache-Control` HTTP response header (e.g., `Cache-Control: max-age=3600`) set by the origin tells CDN edge servers (and browsers) how long they may serve cached content before revalidating with the origin.

**10. Why might a global video streaming platform pre-push popular content to edge servers rather than waiting for the first request in each region to trigger caching?**
Waiting for an organic cache miss means the very first users in each region experience full origin latency and add load to the origin at the moment of a launch, which can be problematic for a high-demand release (e.g., a new episode). Pre-pushing (pre-warming) popular content ensures every region already has a warm cache the moment demand arrives, avoiding a stampede of simultaneous cache misses.

**11. A user in Brazil and a user in Japan both request the same static image from your CDN-backed site. Will they be served from the same edge server? Why or why not?**
No — each user is routed to their own nearest edge server (via DNS-based or Anycast routing), so the Brazilian user is served from a PoP near Brazil and the Japanese user from a PoP near Japan, even though the underlying content and origin are identical. Each edge server independently caches its own copy after its first cache miss.

**12. How does a CDN interact with the caching strategies discussed in the previous video (e.g., cache-aside, TTL-based invalidation)?**
A CDN is essentially a specialized, geographically distributed cache-aside layer for HTTP content: edge servers check their local cache first, and on a miss fetch from the origin (analogous to querying the database) and populate the cache. It uses the same core invalidation concepts — TTLs via Cache-Control headers and explicit purges — just applied at a global, network-edge scale instead of inside a single application.
