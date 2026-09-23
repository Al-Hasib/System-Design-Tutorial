# Module 12: Case Studies — System Design Interview Practice

এটা কোর্সের capstone module। প্রতিটা video একটা classic "design X" interview prompt নিয়ে একটা সম্পূর্ণ system design interview শুরু থেকে শেষ পর্যন্ত walk through করে — requirements clarify করা, capacity estimate করা, একটা high-level architecture আঁকা, এবং সবচেয়ে কঠিন component-গুলোতে deep-dive করা — Module 1-11-এ শেখানো building block-গুলো সরাসরি apply করে (load balancing, sharding, caching, consistent hashing, message queue, CAP theorem, consensus, microservices pattern, transport protocol, message format, security, storage engine internals, distributed coordination, এবং observability/production operations)। এই module শেষ হওয়ার মধ্যে আপনি এই সমস্যাগুলোর যেকোনোটার জন্য একটা পুরো system design interview answer structure ও defend করতে পারবেন, এবং একই method আগে না দেখা prompt-এও adapt করতে পারবেন।

## Videos

| # | Title | Description | Link |
|---|-------|-------------|------|
| 48 | Design a URL Shortener | Key generation, redirect, এবং read-heavy scaling covered করা একটা classic warm-up interview problem। | [48-design-a-url-shortener](./48-design-a-url-shortener/README.md) |
| 49 | Design a Rate Limiter (Practical System Design) | Module 6-এর algorithm ব্যবহার করে একটা production-grade, distributed rate limiter বানানো। | [49-design-a-rate-limiter](./49-design-a-rate-limiter/README.md) |
| 50 | Design a Chat Application (like WhatsApp) | Delivery guarantee, presence, এবং offline sync সহ scale-এ real-time messaging। | [50-design-a-chat-application-whatsapp](./50-design-a-chat-application-whatsapp/README.md) |
| 51 | Design a News Feed System (like Twitter/Facebook) | Feed generation-এ fan-out strategy, ranking, এবং celebrity/hot-key সমস্যা। | [51-design-a-news-feed-system-twitter](./51-design-a-news-feed-system-twitter/README.md) |
| 52 | Design a Distributed File Storage System (like Google Drive/Dropbox) | Cloud file storage-এর জন্য chunking, metadata management, sync, এবং consistency। | [52-design-a-distributed-file-storage-google-drive](./52-design-a-distributed-file-storage-google-drive/README.md) |
| 53 | Design a Video Streaming Platform (like YouTube/Netflix) | Video ingestion, transcoding pipeline, এবং CDN-এর মাধ্যমে adaptive-bitrate delivery। | [53-design-a-video-streaming-platform-youtube-netflix](./53-design-a-video-streaming-platform-youtube-netflix/README.md) |
| 54 | Design a Ride-Sharing System (like Uber) | Scale-এ geospatial indexing, real-time matching, এবং location tracking। | [54-design-a-ride-sharing-system-uber](./54-design-a-ride-sharing-system-uber/README.md) |
