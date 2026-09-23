# অধ্যয়ন নোট: Message Queues, Kafka vs RabbitMQ

## মূল সংজ্ঞা

- **Message queue**: একটি মধ্যস্থতাকারী যা producer-দের পাঠানো message সংরক্ষণ করে যতক্ষণ না consumer-রা সেগুলো প্রসেস করতে প্রস্তুত হয়, যা asynchronous communication সক্ষম করে।
- **Producer**: একটি সার্ভিস/ক্লায়েন্ট যা message তৈরি করে এবং পাঠায়।
- **Broker**: যে সার্ভার(গুলো) message গ্রহণ করে, সংরক্ষণ করে এবং রুট করে।
- **Consumer**: একটি সার্ভিস/ক্লায়েন্ট যা message পড়ে এবং প্রসেস করে।
- **Acknowledgment (ack)**: consumer থেকে broker-কে পাঠানো একটি সিগন্যাল যা নিশ্চিত করে যে message প্রসেস হয়েছে, যা নিরাপদে সরানো বা offset এগিয়ে নেওয়া সম্ভব করে।
- **Backpressure**: queue দ্বারা consumption-এর চেয়ে দ্রুত production-এর burst শুষে নেওয়া, যা ডাউনস্ট্রিম সিস্টেমকে overload থেকে রক্ষা করে।
- **Consumer group**: consumer instance-দের একটি সেট যারা একটি queue/topic consume করার কাজ ভাগ করে নেয়, যা অনুভূমিক (horizontal) স্কেলিং সক্ষম করে।
- **Idempotency**: consumer logic এমনভাবে ডিজাইন করা যাতে একই message দুইবার প্রসেস করলে একবার প্রসেস করার মতোই ফলাফল আসে।

## কেন Asynchronous Messaging?

- **সময়ের** দিক থেকে decouple করে: message পাঠানোর সময় consumer-এর অনলাইনে থাকার দরকার নেই।
- **স্থানের** দিক থেকে decouple করে: producer-এর জানার দরকার নেই কে/কতজন consumer আছে।
- **গতির** দিক থেকে decouple করে: একটি ধীরগতির consumer producer-কে ব্লক করে না বা ধীর করে না।
- Cascading failure প্রতিরোধ করে: একটি dependency বন্ধ থাকলে পুরো request চেইন ভেঙে পড়ে না।
- Producer এবং consumer-দের স্বাধীনভাবে স্কেল করতে সক্ষম করে।

## Kafka vs RabbitMQ

| দিক | RabbitMQ | Kafka |
|---|---|---|
| মডেল | প্রথাগত message broker (smart broker, dumb consumer) | Distributed append-only log (dumb broker, smart consumer) |
| Message-এর জীবনচক্র | Consumption + ack-এর পরে মুছে যায় | Consumption নির্বিশেষে একটি নির্ধারিত সময়ের জন্য সংরক্ষিত থাকে |
| Replay | স্বাভাবিকভাবে সমর্থিত নয় | নেটিভ — consumer-রা offset রিসেট করে rewind/replay করতে পারে |
| Routing | Exchange-এর মাধ্যমে সমৃদ্ধ routing (direct, topic, fanout, headers) | সরল — topic + partition ভিত্তিক |
| Throughput | উচ্চ, কিন্তু চরম স্কেলে সাধারণত Kafka-র চেয়ে কম | অত্যন্ত উচ্চ throughput, লক্ষ লক্ষ event/সেকেন্ডের জন্য নির্মিত |
| Ordering | Per-queue ordering | Per-partition ordering গ্যারান্টি |
| একই ডেটার একাধিক স্বতন্ত্র consumer | কঠিন (consume হওয়ার পরে message সরে যায়) | স্বাভাবিক — প্রতিটি consumer group নিজের offset ট্র্যাক করে |
| সবচেয়ে উপযুক্ত | Task queue, RPC-স্টাইল workflow, জটিল routing | Event streaming, event sourcing, log aggregation, analytics pipeline |
| প্রোটোকল | AMQP (MQTT, STOMP-ও সমর্থন করে) | TCP-এর উপর কাস্টম বাইনারি প্রোটোকল |
| Operational জটিলতা | কম থেকে মাঝারি | বেশি (ZooKeeper/KRaft, partition ব্যবস্থাপনা প্রয়োজন) |

## Delivery Guarantees

| Guarantee | বর্ণনা | ঝুঁকি |
|---|---|---|
| At-most-once | Message একবার পাঠানো হয়, কোনো retry নেই | Message হারিয়ে যাওয়া সম্ভব |
| At-least-once | Acknowledge না হওয়া পর্যন্ত broker retry করে | Duplicate প্রসেসিং সম্ভব |
| Exactly-once | প্রতিটি message ঠিক একবার প্রসেস হয় | সিস্টেমের সীমানা জুড়ে অর্জন করা কঠিন; Kafka তার নিজের pipeline-এর মধ্যে idempotent producers + transactions-এর মাধ্যমে এটি সমর্থন করে |

- বেশিরভাগ বাস্তব সিস্টেমে ব্যবহারিক ডিফল্ট: **at-least-once delivery + idempotent consumers**।

## স্কেলিং প্রক্রিয়া

- **Partitioning** (Kafka): একটি topic-কে broker জুড়ে বিতরণ করা partitions-এ ভাগ করা হয়; প্রতিটি partition একটি ordered, append-only log।
- **Consumer groups**: একটি group-এর মধ্যে, প্রতিটি partition একবারে ঠিক একটি consumer instance দ্বারা consume হয় — এভাবেই Kafka consumption সমান্তরাল (parallelize) করে।
- **Prefetch / QoS** (RabbitMQ): একজন consumer কতগুলো unacknowledged message ধরে রাখতে পারে তা নিয়ন্ত্রণ করে, throughput এবং fairness-এর মধ্যে ভারসাম্য রক্ষা করে।

## দ্রুত সারসংক্ষেপ

- যখন সার্ভিসগুলোর মধ্যে নির্ভরযোগ্য, decoupled, asynchronous communication দরকার তখন একটি queue ব্যবহার করুন।
- জটিল routing এবং ক্লাসিক task/job queue-এর জন্য RabbitMQ বেছে নিন।
- উচ্চ-থ্রুপুট event streaming, replay, এবং একই ডেটা পড়া একাধিক স্বতন্ত্র consumer অ্যাপ্লিকেশনের জন্য Kafka বেছে নিন।
- সবসময় consumer-দের idempotent হিসেবে ডিজাইন করুন — production সিস্টেমে at-least-once delivery ধরে নিন।
</content>
