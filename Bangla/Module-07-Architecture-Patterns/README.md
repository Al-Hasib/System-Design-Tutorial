# Module 7: Architecture Patterns

এই module individual building block থেকে zoom out করে পুরো system-এর shape-এর দিকে তাকায়। আপনি শিখবেন কীভাবে monolith বনাম microservices-এর মধ্যে decide করবেন, independently deploy করা service-গুলো runtime-এ কীভাবে নির্ভরযোগ্যভাবে একে অপরকে খুঁজে পায় ও কথা বলে, এবং Domain-Driven Design কীভাবে technical layer-এর বদলে business capability-কে কেন্দ্র করে service boundary আঁকার একটা principled উপায় দেয়। এই video-গুলো মিলে আগের module-গুলোর low-level primitive (load balancer, message queue, database)-কে real system এবং সেগুলোর মালিক team-গুলো structure করার একটা organizational ও architectural strategy-তে যুক্ত করে।

## এই Module-এর Video সমূহ

| # | Title | Description | Link |
|---|-------|-------------|------|
| 30 | Monolith vs Microservices | দুইটা প্রধান architectural style তুলনা করা হয়েছে, তাদের trade-off, এবং আপনার team ও product stage-এর জন্য কোনটা মানানসই তা কীভাবে ঠিক করবেন। | [30-monolith-vs-microservices](./30-monolith-vs-microservices/README.md) |
| 31 | Microservices Communication & Service Discovery | Synchronous বনাম asynchronous inter-service communication এবং service-গুলো runtime-এ dynamically কীভাবে একে অপরকে খুঁজে পায় তা ব্যাখ্যা করা হয়েছে। | [31-microservices-communication-and-service-discovery](./31-microservices-communication-and-service-discovery/README.md) |
| 32 | Domain-Driven Design Basics for System Design | Microservice boundary ভাগ করার একটা method হিসেবে bounded context এবং ubiquitous language-এর মতো DDD concept পরিচয় করানো হয়েছে। | [32-domain-driven-design-basics](./32-domain-driven-design-basics/README.md) |
