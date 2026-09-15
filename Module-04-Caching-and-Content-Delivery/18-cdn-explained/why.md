# Why This Topic Matters: CDN (Content Delivery Network)

> **In one sentence:** No amount of backend optimization can beat the speed of light, so if your users are far from your servers, the only real fix is to move the content closer to them.

## The World Before This Idea

Your servers are in Virginia. A user in Sydney loads your homepage. The round trip is roughly 200 ms — a hard physical floor, not a tuning problem. The page needs 60 resources: HTML, CSS, JavaScript bundles, fonts, and images. Even with connection reuse and parallelism, that's several seconds of pure network waiting before anything renders.

Now multiply. A 2 MB page times 10 million requests is 20 TB of egress, all of it from your origin, all of it billed at origin rates, all of it competing with your dynamic API traffic for the same bandwidth. And when a video goes viral, that traffic arrives all at once, at a single location.

## The Problems It Solves

### 1. Distance you cannot optimize away
**What you see:** Your app feels instant in the office and sluggish for half your users, and profiling shows the backend is fast.

**Why it happens:** Latency is dominated by propagation delay. Sydney to Virginia is about 16,000 km; light in fiber covers that in roughly 80 ms one way, and real routing makes it worse. No server upgrade touches this number.

**How a CDN solves it:** Content is cached on edge servers in hundreds of cities. The Sydney user is served from Sydney — single-digit milliseconds instead of 200+. This is the only mechanism that actually reduces the distance.

### 2. Origin bandwidth and cost
**What you see:** Enormous egress bills and an origin saturated by static file serving rather than by anything valuable.

**Why it happens:** Every byte of every image, script, and video is served from one place, repeatedly, identically.

**How a CDN solves it:** The edge absorbs 90-99% of traffic. Your origin serves each asset once per edge location per cache period. Egress costs drop sharply (CDN bandwidth is cheaper at volume) and your servers spend their CPU on requests that actually require them.

### 3. Traffic spikes that flatten the origin
**What you see:** A product launch, a viral post, or a marketing email lands and the site goes down under load you never provisioned for.

**Why it happens:** All traffic converges on fixed origin capacity.

**How a CDN solves it:** The edge network has vastly more aggregate capacity than you will ever provision, and a spike for cacheable content is absorbed there. This is also the first layer of DDoS protection — volumetric attacks hit a network built to absorb them instead of your origin.

### 4. Video and large files that can't be served conventionally
**What you see:** Streaming video from your origin buffers constantly and costs a fortune.

**Why it happens:** Video is enormous, bandwidth-intensive, and extremely latency-sensitive for startup time.

**How a CDN solves it:** Edge caching of video segments is precisely what makes streaming viable. Netflix and YouTube are, architecturally, CDN companies with a catalog attached — this isn't an optimization for them, it's the core design.

## The Price You Pay

- **Invalidation across hundreds of nodes.** You pushed a bad CSS file with a one-year cache header; it's now cached in 300 cities. Purges exist but propagate unevenly. The standard mitigation is content-hashed filenames (`app.a3f9c2.js`) so new content gets a new URL and old content simply stops being requested — worth knowing because it's the fix, not a workaround.
- **Stale content confusion.** Users on different continents can see different versions of your site for a period after deploy.
- **Dynamic and personalized content doesn't cache trivially.** Anything user-specific must either bypass the cache or use careful cache keys — and a cache-key mistake that serves one user's personalized page to another is a serious data leak, not just a bug.
- **Cost and complexity.** Another vendor, another config surface, another thing in the request path that can misbehave. CDN misconfiguration causes real outages.
- **Debugging.** Reproducing a problem that only occurs at one edge location, on one cached variant, is genuinely difficult.

## When You Need It — and When You Don't

| Use a CDN when | Skip it when |
|---|---|
| You serve static assets (JS, CSS, images, fonts, video) | It's an internal tool with users in one office |
| Your users are geographically distributed | All traffic is dynamic and uncacheable |
| Bandwidth costs are significant | Traffic is low enough that origin serving is trivially cheap |
| You need DDoS absorption at the edge | — |
| You're serving media at scale | — |

## Why This Shows Up in Interviews

Any design involving images, video, or a global user base should include a CDN, and omitting one in a YouTube or Netflix design is a notable miss. Beyond placing it on the diagram, interviewers look for: what's cacheable versus what must hit origin, how you invalidate (and the content-hashing trick), what cache headers you set, and whether you understand that dynamic personalized content needs different handling. In video designs, understanding that the CDN is the core of the architecture — not an add-on — is the key insight.

## How It Connects

A CDN is **caching** (topic 17) applied at the network edge, and each edge node is effectively a geographically distributed **reverse proxy**. It's the cheapest form of **multi-region architecture** — global presence for static content without running global infrastructure. It depends on **HTTP** cache-control semantics to work at all, and it's a central component of the **video streaming** and **file storage** case studies.

**Next:** [Distributed Caching with Redis & Memcached](../19-distributed-caching-redis-and-memcached/why.md) — the caching layer inside your own infrastructure.
