# Module 5: Messaging & Asynchronous Systems

এই module-এ দেখানো হয়েছে বড় scale-এর system কীভাবে সরাসরি একে অপরের সাথে কথা না বলেও communicate করে: message queue, publish-subscribe fan-out, event-driven architecture, এবং batch ও stream data processing-এর মধ্যে trade-off। এই pattern-গুলো service-কে time এবং space-এ decouple করতে দেয় — একটা producer-এর দরকার হয় না consumer online, দ্রুত, বা এমনকি তার অস্তিত্ব সম্পর্কে জানুক — এই কারণেই system traffic spike, partial failure, এবং independent team deployment সহ্য করতে পারে। এই module ভালোভাবে আয়ত্ত করা জরুরি এমন system design করার জন্য যা brittle synchronous call-এর জটে না জড়িয়ে horizontally scale করতে পারে।

## এই Module-এর Video সমূহ

| # | Title | Description | Link |
|---|-------|-------------|------|
| 20 | Message Queues Explained: Kafka vs RabbitMQ | Message queue কীভাবে producer এবং consumer-কে decouple করে, এবং কখন Kafka বনাম RabbitMQ বেছে নেবেন | [20-message-queues-kafka-vs-rabbitmq](./20-message-queues-kafka-vs-rabbitmq/README.md) |
| 21 | Publish-Subscribe Pattern | Pub-sub কীভাবে একটা event একাধিক independent subscriber-এর কাছে fan out করে | [21-publish-subscribe-pattern](./21-publish-subscribe-pattern/README.md) |
| 22 | Event-Driven Architecture | Direct call-এর বদলে event-কে কেন্দ্র করে system design করা | [22-event-driven-architecture](./22-event-driven-architecture/README.md) |
| 23 | Batch Processing vs Stream Processing | Data batch-এ প্রসেস করবেন নাকি continuous stream হিসেবে, তা বেছে নেওয়া | [23-batch-vs-stream-processing](./23-batch-vs-stream-processing/README.md) |
