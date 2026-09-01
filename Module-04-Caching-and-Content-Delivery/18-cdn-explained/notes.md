# Study Notes: CDN (Content Delivery Network) Explained

## Definitions

- **CDN (Content Delivery Network)**: A globally distributed system of servers that caches and delivers content from locations physically close to end users.
- **Origin server**: The authoritative server where the real, canonical copy of the content lives.
- **Edge server**: A CDN server positioned close to end users, caching copies of origin content.
- **PoP (Point of Presence)**: A physical data center location where a CDN operates one or more edge servers.
- **Cache-Control headers**: HTTP response headers (e.g., `Cache-Control: max-age=3600`) that tell caches (browsers, CDNs) how long to keep content before revalidating.
- **Purge/Invalidation**: An explicit action to remove cached content from edge servers before its TTL expires.
- **Anycast**: A routing technique where the same IP address is announced from multiple physical locations; network routing (BGP) sends traffic to the topologically nearest one.
- **Edge computing**: Running application logic directly on CDN edge servers rather than only at the origin.

## Static vs Dynamic Content Caching

| Aspect | Static Content | Dynamic Content |
|---|---|---|
| Examples | Images, CSS, JS bundles, videos, downloadable files, static HTML | Personalized pages, API responses, search results |
| Cacheability | Same response for all users — easy to cache | Varies per user/request — harder to cache safely |
| Typical TTL | Minutes to days (often long, with cache-busting filenames for updates) | Seconds (micro-caching) or not cached at all |
| Modern techniques | Standard edge caching | Edge computing, micro-caching, cache keys based on headers/cookies |

## Request Routing Techniques

| Technique | How it works | Pros | Cons |
|---|---|---|---|
| DNS-based routing | DNS resolver returns different IPs based on the geographic/network location of the query | Simple, widely supported | DNS caching/resolver location can reduce accuracy; slower failover |
| Anycast (BGP) | Same IP announced from many locations; internet routing sends traffic to the "nearest" announcer | Fast, automatic failover, no DNS trickery | Requires BGP-level infrastructure; more complex to operate |

## CDN Invalidation Approaches

| Method | Description |
|---|---|
| TTL / Cache-Control headers | Origin specifies how long edge servers may cache a response before revalidating |
| Manual Purge | API/dashboard action to immediately evict a cached object across all edge servers |
| Versioned/cache-busting URLs | Change the filename/URL (e.g., append a hash) so a "new" resource is fetched fresh, avoiding invalidation entirely |

## Key Numbers / Rules of Thumb

- Speed of light in fiber is roughly 200,000 km/s; a round trip across ~15,000 km (e.g., US to Singapore) alone costs on the order of 100+ ms before any server processing.
- CDNs can reduce time-to-first-byte for geographically distant users from hundreds of milliseconds down to tens of milliseconds by serving from a nearby edge.
- Large CDN providers operate hundreds to thousands of PoPs across the globe.
- Well-cached static assets can achieve cache hit ratios above 90% at the edge, dramatically reducing origin load.

## Summary

- CDNs cache content physically close to users to minimize latency caused by network distance.
- Origin holds the source of truth; edge servers at PoPs hold cached copies.
- Static content is the easy win; dynamic content requires edge computing or micro-caching to benefit safely.
- Routing to the nearest edge server uses DNS-based routing or Anycast.
- CDN caches still need invalidation strategies (TTL, purge, cache-busting URLs) just like any other cache.
- CDNs double as performance tools and infrastructure protection (traffic spikes, DDoS absorption).
