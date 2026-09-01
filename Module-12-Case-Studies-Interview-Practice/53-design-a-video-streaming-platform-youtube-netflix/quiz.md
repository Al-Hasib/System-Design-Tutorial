# Quiz: Follow-up Interview Questions

Practice questions an interviewer might ask after the initial design in [README.md](README.md). Try answering each yourself before reading the model answer.

**1. A video suddenly goes viral and traffic spikes 100x in minutes. How do you handle it?**
Rely on the CDN's request coalescing so concurrent cache misses at an edge collapse into a single origin fetch instead of stampeding origin storage. Use view-velocity signals to proactively pre-warm additional edge caches and add an origin-shield layer between edges and durable storage to absorb fan-in. Autoscale any stateless read-path services (metadata API) behind the load balancer.

**2. How do you reduce transcoding cost without hurting user experience?**
Don't pre-generate every rendition/codec combination for every video. Pre-transcode only the most commonly requested renditions (e.g., 360p/720p in H.264) at upload time, and generate rarer ones (4K, AV1) lazily on first request, caching the result afterward. Also prioritize the transcoding queue so popular or trending uploads get processed first.

**3. How would you support live streaming in addition to video-on-demand (VOD)?**
Live streaming trades some of VOD's guarantees for very low end-to-end latency: instead of transcoding a full file, you transcode a continuous stream in small segments (a few seconds each) as it arrives, publish an ever-growing manifest, and push segments to CDN edges immediately. There's no "raw master" to re-transcode later, retry budgets are tighter, and protocols often favor low-latency HLS/DASH extensions or WebRTC for sub-second latency use cases.

**4. How do you decide which renditions to pre-generate versus generate on demand?**
Base it on observed request distribution: instrument which resolutions/codecs are actually requested per device/network mix, and pre-generate the renditions that cover the bulk of playback requests (often 360p/480p/720p H.264 covers the majority). Long-tail renditions (4K, legacy codecs, unusual aspect ratios) are generated on-demand and cached, since most videos are never viewed at those settings.

**5. How do you handle upload resumability for large video files?**
Split the upload into fixed-size chunks with a resumable upload protocol (e.g., an upload session ID plus per-chunk checksums/offsets, similar to the S3 multipart upload API or the tus protocol). If the connection drops, the client queries which chunks the server already has and resumes from the next missing chunk instead of restarting the whole file.

**6. What DRM / content protection considerations matter for a platform like Netflix?**
Licensed content typically requires encrypting segments (e.g., via AES-128 for HLS or Common Encryption/CENC for DASH) and integrating with DRM systems (Widevine, PlayReady, FairPlay) so only authorized, licensed players can decrypt and play back content. This adds a license-server round trip before playback and requires key rotation strategies, but is a hard requirement for licensed/premium content rather than user-generated content.

**7. How do you keep view counters accurate without creating a write bottleneck?**
Don't increment a row in the primary metadata store synchronously per view. Instead, emit view events to a streaming pipeline, aggregate them in time windows (e.g., every few seconds), and flush batched increments to the sharded metadata store. This trades a small amount of counter staleness for orders-of-magnitude fewer writes on hot videos.

**8. How would you reduce startup latency (time to first frame)?**
Keep initial segments small so the player only needs to download a few seconds of a conservative (low/medium) bitrate before playback starts. Serve the manifest and first segments from the nearest CDN edge, and consider predictive pre-fetching (e.g., pre-fetch manifest and first segment when a user hovers over/selects a video) to hide network round-trip time.

**9. How do you choose between H.264, VP9, and AV1 for encoding?**
H.264 has universal hardware decode support, so it's a required baseline for compatibility, especially on older devices. VP9 and AV1 compress significantly better (lower bitrate at equal quality), reducing storage and CDN egress cost at scale, but cost more CPU to encode and have less universal hardware decode support. Platforms with massive egress costs (like YouTube/Netflix) encode in multiple codecs and serve the best one the client supports.

**10. How does database sharding strategy affect metadata queries like "list all videos by a channel"?**
If you shard purely by `videoId` hash, listing all videos for a channel requires a scatter-gather query across shards, which is slow. A common mitigation is to maintain a secondary index or a separate mapping table (channelId -> list of videoIds) that's itself sharded by `channelId`, so channel-scoped queries hit a single shard, while individual video lookups by `videoId` use the primary sharding scheme.

**11. What happens if a transcoding worker crashes mid-job?**
Because the pipeline is queue-driven, the job message isn't acknowledged/committed until transcoding and storage writes succeed. If a worker crashes, the message becomes visible again (via queue visibility timeout or consumer group rebalance) and another worker picks it up. Transcoding should be idempotent (writing to a unique output path, or overwriting deterministically) so a retried job doesn't corrupt state.

**12. How would you support multiple audio tracks or subtitles?**
Treat them as additional, independently-encoded and independently-cached streams referenced by the same manifest — HLS/DASH manifests natively support multiple audio/subtitle tracks alongside video renditions, and the player selects/switches tracks the same way it switches video bitrate, without needing a separate delivery pipeline.
