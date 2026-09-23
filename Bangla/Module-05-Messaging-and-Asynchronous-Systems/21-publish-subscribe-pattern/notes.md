# অধ্যয়ন নোট (Study Notes): Publish-Subscribe Pattern

## মূল সংজ্ঞাসমূহ (Core Definitions)

- **Publish-Subscribe (pub-sub)**: একটা messaging pattern যেখানে publisher-রা একটা topic/channel-এ message broadcast করে, এবং যেকোনো সংখ্যক স্বাধীন subscriber প্রতিটা message-এর নিজস্ব একটা copy পায়।
- **Publisher**: একটা topic-এ message পাঠায়, কে (যদি কেউ থাকে) subscribe করেছে সে সম্পর্কে কোনো ধারণা ছাড়াই।
- **Subscriber**: একটা topic-এ আগ্রহ register করে এবং সেখানে publish হওয়া প্রতিটা message-এর একটা copy পায়।
- **Topic / Channel**: নামযুক্ত গন্তব্য যেখানে publisher-রা লেখে এবং subscriber-রা পড়ে।
- **Fan-out**: একটা publish করা message একইসাথে একাধিক subscriber-এর কাছে পৌঁছে দেওয়ার প্রক্রিয়া।

## Queue (Point-to-Point) বনাম Pub-Sub

| দিক | Point-to-Point Queue | Publish-Subscribe |
|---|---|---|
| Delivery | একটা message ঠিক একজন consumer (অথবা প্রতি consumer group-এ একজন) consume করে | একটা message প্রতিটা subscriber-এর কাছে স্বাধীনভাবে পৌঁছায় |
| ব্যবহারের ক্ষেত্র | worker-দের মধ্যে বিচ্ছিন্ন কাজ/job বণ্টন করা | অনেক আগ্রহী পক্ষের কাছে একটা event/notification broadcast করা |
| Coupling | Producer হয়তো চায় যে কাজটা একবার হোক | Producer-এর subscriber-এর সংখ্যা বা পরিচয় সম্পর্কে কোনো ধারণা থাকে না |
| Scaling model | load ভাগ করতে আরও consumer যোগ করুন | একই event-এ স্বাধীনভাবে সাড়া দিতে আরও subscriber যোগ করুন |
| উদাহরণ | Image resizing job queue | "user signed up" event যা email, analytics, loyalty service-কে জানায় |

## সাধারণ Pub-Sub Implementation

| Technology | Durability | Delivery Guarantee | সাধারণ ব্যবহার |
|---|---|---|---|
| Redis Pub/Sub | নেই (in-memory, fire-and-forget) | At-most-once; subscriber অফলাইন থাকলে miss হয় | Real-time, low-stakes notification |
| Apache Kafka | Durable log, replayable | At-least-once (configurable, Kafka-র মধ্যে exactly-once) | High-throughput event streaming, একাধিক স্বাধীন consumer group |
| AWS SNS | Durable, প্রায়ই প্রতি subscriber-এর জন্য SQS-এর সাথে জোড় করা | At-least-once | একাধিক AWS service/queue-তে fan-out |
| Google Cloud Pub/Sub | Durable | At-least-once | Managed, global-scale event distribution |
| Azure Service Bus Topics | Durable | At-least-once | subscription filter সহ enterprise messaging |
| MQTT | QoS level (0/1/2) অনুযায়ী ভিন্ন | QoS অনুযায়ী at-most-once থেকে exactly-once | IoT, low-bandwidth sensor network |

## বড় Scale-এ Pub-Sub কেন গুরুত্বপূর্ণ

- **Loose coupling** সম্ভব করে: নতুন subscriber যোগ হলে publisher-কে পরিবর্তন করতে হয় না।
- **Team autonomy** সম্ভব করে: বিভিন্ন team স্বাধীনভাবে একই event stream-এ subscribe করতে পারে।
- **one-to-many** communication pattern স্বাভাবিকভাবে সমর্থন করে, যা point-to-point queue করে না।

## Tradeoff / ঝুঁকি

- **Ordering** globally guarantee করা কঠিন হতে পারে, বিশেষ করে একাধিক publisher বা partitioned topic থাকলে।
- **Delivery guarantee পরিবর্তিত হয়** — কিছু implementation কোনো subscriber না শুনলে message ফেলে দেয় (fire-and-forget); অন্যগুলো persist করে এবং at-least-once guarantee দেয়।
- **Discoverability/debugging কঠিন**: publisher জানে না তার event কে consume করছে, তাই একটা সিস্টেমজুড়ে প্রভাব ট্রেস করতে ভালো documentation, schema, এবং observability দরকার।
- শক্তিশালী event contract ছাড়া একটা implicit, ট্রেস-করা-কঠিন dependency graph ("distributed monolith")-এর ঝুঁকি।

## দ্রুত সারাংশ (Quick Summary)

- pub-sub ব্যবহার করুন যখন একটা event-কে একাধিক স্বাধীন consumer-এর কাছে পৌঁছাতে হবে যাদের একে অপরের সাথে বা publisher-এর সাথে coupled হওয়া উচিত নয়।
- একটা point-to-point queue ব্যবহার করুন যখন আপনার প্রতিটা কাজের একক ঠিক একজন worker-এর হাতে handle করা দরকার।
- শুধু "pub-sub বনাম queue" পরিভাষার ভিত্তিতে নয়, বরং durability এবং delivery-guarantee প্রয়োজনীয়তার ভিত্তিতে underlying technology বেছে নিন।
</content>
