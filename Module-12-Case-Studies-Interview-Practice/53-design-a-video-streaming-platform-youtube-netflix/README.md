# Design a Video Streaming Platform (like YouTube/Netflix)

**Difficulty:** Advanced (Capstone). **Estimated length:** 25-30 min. **Prerequisites:** [Database Sharding and Partitioning](../../Module-03-Databases-and-Storage/14-database-sharding-and-partitioning/README.md), [Caching Strategies and Cache Invalidation](../../Module-04-Caching-and-Content-Delivery/17-caching-strategies-and-cache-invalidation/README.md), [CDN Explained](../../Module-04-Caching-and-Content-Delivery/18-cdn-explained/README.md), [Message Queues: Kafka vs RabbitMQ](../../Module-05-Messaging-and-Asynchronous-Systems/20-message-queues-kafka-vs-rabbitmq/README.md), [Batch vs Stream Processing](../../Module-05-Messaging-and-Asynchronous-Systems/23-batch-vs-stream-processing/README.md), [Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md), [Scalability Basics: Vertical vs Horizontal Scaling](../../Module-01-Foundations/04-scalability-basics-vertical-vs-horizontal-scaling/README.md)

## Learning Objectives

- Structure a system design interview for a large-scale video streaming platform from requirements to trade-offs.
- Estimate storage, transcoding, and CDN egress bandwidth at YouTube/Netflix scale.
- Design an asynchronous, queue-driven transcoding pipeline that decouples upload from processing.
- Explain how CDNs, edge caching, and adaptive bitrate streaming (HLS/DASH) combine to deliver low-latency, high-availability playback globally.
- Identify the major bottlenecks in video platforms (viral thundering herd, long-tail storage cost, transcoding cost/latency) and reason about trade-offs.

## Script

### Hook / Intro

Every minute, hundreds of hours of video get uploaded to platforms like YouTube. Every one of those uploads has to be transcoded into half a dozen formats, stored durably, and made instantly streamable to billions of devices worldwide — phones on 3G, smart TVs on gigabit fiber, laptops on airport Wi-Fi — without buffering. This is one of the most popular capstone interview questions because it touches almost every module in this course: storage and sharding, caching and CDNs, message queues, load balancing, and scalability fundamentals all come together in one system. Today we're going to design a video streaming platform like YouTube or Netflix, end to end, the way you'd walk through it in a real onsite interview.

### Step 1: Clarify Requirements

Before drawing a single box, we clarify scope with the interviewer.

**Functional requirements:**
- Users can **upload** a video file of varying size and format.
- The platform **transcodes** each upload into multiple resolutions and bitrates (e.g., 240p, 360p, 480p, 720p, 1080p, 4K) so clients can adapt to their network conditions.
- Users can **stream/play back** videos with **adaptive bitrate streaming** — the player switches quality seamlessly as bandwidth changes.
- Basic metadata operations: title, description, thumbnail, view count, likes.

**Explicitly out of scope** (call this out loudly in an interview — it shows judgment): full-text search ranking, recommendation systems, comments/social graph, live streaming ingest (we'll touch on it briefly in trade-offs), monetization/ads. We'll mention these exist but focus our design time on upload → transcode → store → deliver → play.

**Non-functional requirements:**
- **High availability** — video playback is the core product; the read path must survive regional failures.
- **Low startup latency and minimal buffering** — users expect playback to start in under 1-2 seconds and rebuffer rarely.
- **Massive read-heavy scale** — reads (views) vastly outnumber writes (uploads), maybe by a factor of 1000:1 or more.
- **Durability** — once a video is accepted, it must never be lost; we're fine with eventual consistency on view counters, but the actual video bytes need strong durability guarantees (e.g., 11 nines, similar to object storage like S3).

### Step 2: Capacity Estimation

Let's put real numbers on this, YouTube-scale.

**Uploads:** Assume ~500 hours of video uploaded per minute. That's 500 × 60 = 30,000 video-hours/day. At an average bitrate of ~5 Mbps for the raw source, one hour of video is roughly 2.25 GB. So raw upload volume ≈ 30,000 × 2.25 GB ≈ **67 TB/day of raw ingested video**.

**Transcoding fan-out:** Each uploaded video gets transcoded into ~5-6 renditions (240p through 1080p/4K, each in one or two codecs like H.264 and AV1/VP9). If the average rendition set adds up to roughly 1.5-2x the raw size (lower resolutions are much smaller, but 4K and multiple codecs add back weight), we're storing on the order of **100-150 TB/day of encoded output**, in addition to keeping the raw/master copy for future re-encodes. Over a year, that's tens of petabytes — this is why platforms use tiered, lifecycle-managed object storage.

**Read traffic:** Assume ~5 billion video views/day globally. That's roughly 5,000,000,000 / 86,400 ≈ **58,000 views/second average**, with peak traffic (evenings, viral events) hitting 3-5x that, so **~150,000-250,000 concurrent stream starts/second at peak**.

**CDN egress bandwidth:** If the average concurrently-streaming session consumes ~3 Mbps (mix of mobile 480p and desktop 1080p), and at peak we have, say, 50-100 million concurrent viewers globally, egress bandwidth is 50,000,000 × 3 Mbps = **150 Tbps** at global peak (Netflix has publicly cited numbers in this range during peak hours). Even a mid-size platform serving 1 million concurrent streams at 3 Mbps needs **3 Tbps** of egress — no single data center provides that; it has to come from a CDN with edge PoPs distributed worldwide.

These numbers justify every architectural decision that follows: we need object storage with lifecycle tiers, an asynchronous fan-out transcoding pipeline, and a CDN-first delivery model — we cannot serve video directly from origin servers at this scale.

### Step 3: High-Level Design

At a high level, the pipeline looks like:

**Upload Service → Raw Storage → Transcoding Pipeline (queue-driven, distributed workers) → Encoded Storage / Object Store → Metadata Service → CDN → Client Adaptive Bitrate Player**

Walking through it: a client uploads a video (often in chunks, to support resumable uploads over flaky connections) to an **Upload Service** sitting behind a load balancer (see [Load Balancing Explained](../../Module-02-Networking-and-Communication/07-load-balancing-explained/README.md)). The upload service writes the raw file to **raw/master storage** — durable object storage such as S3 or GCS — and publishes an "upload complete" event.

That event lands on a **message queue** (Kafka), which fans it out to a pool of **distributed transcoding workers**. Each worker pulls the raw file, runs it through an encoder (e.g., FFmpeg-based pipelines), and produces the multiple resolution/bitrate renditions, chunked into small segments for adaptive streaming. Encoded outputs are written to **encoded storage** — again an object store, but now paired with a CDN origin.

Once transcoding finishes, the **Metadata Service** (backed by a sharded database) is updated: video status flips from "processing" to "ready," along with pointers to each rendition's manifest and segment URLs, thumbnails, duration, etc.

For playback, the **client player** first requests the video's manifest (an HLS `.m3u8` or DASH `.mpd` file) from the metadata/API layer, then streams segments directly from the nearest **CDN edge node**, which either has the segments cached or pulls them from origin (encoded storage) on a cache miss. The player continuously measures bandwidth and switches renditions segment-by-segment — this is adaptive bitrate streaming.

This design cleanly separates concerns: uploads and transcoding are write-path, asynchronous, and horizontally scalable; playback is read-path, CDN-dominated, and must be near-instant.

### Step 4: Deep Dive on Key Components

**1. Asynchronous Transcoding Pipeline (Message Queues, Module 5).** We never transcode synchronously during upload — that would tie up the upload connection for minutes and couple two very different scaling needs. Instead, the upload service just writes the raw file and drops a message ("videoId: X, ready to transcode") onto a Kafka topic. A pool of stateless transcoding workers consumes from partitions of that topic, allowing us to scale workers independently of upload traffic and to retry failed jobs without affecting the client. This is a textbook case of **batch vs. stream processing trade-offs**: transcoding a single video is inherently a batch-style, CPU-bound job (we process the whole file, or large chunks of it, at once), while updating view counters and analytics in near-real-time is better suited to a streaming pipeline. We use Kafka as the backbone for both, but with different consumer patterns — batch-style consumer groups for transcoding, and streaming aggregation (e.g., windowed counts) for view metrics.

**2. CDN and Edge Caching (Module 4).** Once a video is encoded, the actual bytes need to reach users with low latency regardless of geography. We push encoded segments to a CDN with edge Points of Presence (PoPs) close to users. Popular videos are cached at nearly every edge node (hot content), so playback requests never hit origin. This is where the **caching strategy** matters: a cache-aside pattern is common, where the CDN edge fetches from origin object storage on a first miss and then serves cached copies for a TTL, with invalidation typically driven by content versioning rather than active purges (since video segments are immutable once encoded).

**3. Adaptive Bitrate Streaming (HLS/DASH).** Each rendition is chunked into small segments (2-10 seconds each). The player downloads a manifest listing all available renditions and their segment URLs, starts with a conservative bitrate, then measures throughput and switches up or down segment-by-segment. This is why startup latency is low — the player only needs the first segment of a low/medium bitrate to start playing, not the whole file.

**4. Database Sharding for Metadata and View Counters (Module 3).** With billions of videos and view events, a single database cannot hold the metadata table or absorb the write rate of view-count increments. We shard the metadata store by `videoId` (hash-based sharding spreads load evenly and avoids hot shards from viral clustering). View counters are a special case: rather than incrementing a row in the primary metadata shard on every view (which would create massive write contention on popular videos), we batch/aggregate view events through the streaming pipeline and periodically flush aggregated counts to the sharded store — trading a few seconds/minutes of counter staleness for orders-of-magnitude less write load.

### Step 5: Bottlenecks & Trade-offs

- **Transcoding cost/latency vs. pre-generating all renditions.** Transcoding every video into every rendition and codec immediately is expensive in compute and delays availability. An alternative is to transcode only the most common renditions up front (e.g., 360p/720p) and lazily generate rarer ones (4K, older codecs) on first request, cached thereafter. This trades a slightly slower first-play for less wasted compute on videos nobody watches in 4K.
- **Hot vs. long-tail content caching and storage cost.** A small fraction of videos (hot/viral content) account for the vast majority of views and should live on nearly every CDN edge. The long tail — videos with only a handful of views — doesn't justify broad caching and can be tiered to cheaper, colder storage classes, accepting higher latency on the rare access.
- **Thundering herd on newly viral videos.** A video that suddenly goes viral can overwhelm origin storage if every edge node simultaneously misses cache and requests the same segments. Mitigations include request coalescing at the edge (collapsing concurrent misses into one origin fetch), pre-warming CDN caches when early view-velocity signals a video is trending, and origin-shield layers that absorb the fan-in before it reaches durable storage.
- **Codec/format trade-offs.** Newer codecs like AV1 or VP9 compress significantly better than H.264, reducing storage and egress cost, but they're more CPU-expensive to encode and have less universal hardware decode support on older devices. Most platforms encode in H.264 for broad compatibility plus AV1/VP9 for supporting newer devices and reducing long-term bandwidth cost.

### Recap

We clarified requirements around upload, transcoding, and adaptive playback; sized the system at hundreds of terabytes of daily storage growth and tens to hundreds of terabits per second of peak CDN egress; designed a pipeline that decouples upload from transcoding via message queues; and pushed delivery to a CDN using adaptive bitrate streaming, backed by a sharded metadata store. We closed with the real-world trade-offs around transcoding cost, caching hot vs. long-tail content, viral thundering herds, and codec choice.

### What's Next

This capstone pulled together storage, caching, CDNs, messaging, and sharding into one system. From here, revisit any of the linked prerequisite modules to go deeper on a specific building block, or explore how live streaming (a fundamentally different, latency-sensitive variant of this problem) changes these trade-offs.

## Key Takeaways

- Separate the write path (upload → async transcoding) from the read path (CDN-driven playback) — they have very different scaling and latency requirements.
- Message queues decouple upload from transcoding, enabling independent scaling and retries; transcoding is a batch-style workload, while view-count aggregation is better suited to streaming.
- CDNs and edge caching are mandatory at this scale — no origin fleet can serve terabits-per-second of egress directly.
- Adaptive bitrate streaming (HLS/DASH) with small, immutable segments enables fast startup and seamless quality switching.
- Shard metadata and batch/aggregate high-frequency writes like view counters to avoid hot-shard contention.
- Real-world trade-offs (pre-transcode vs. on-demand, hot vs. long-tail caching, viral thundering herd, codec choice) are what separate a working design from a scalable one.
