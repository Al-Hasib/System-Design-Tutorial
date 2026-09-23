# অনুশীলন ও ইন্টারভিউ প্রশ্ন

**১. দুটি সার্ভিসকে সরাসরি একে অপরকে কল করতে দেওয়ার (যেমন, REST-এর মাধ্যমে) বদলে কেন তাদের মধ্যে একটি message queue প্রবর্তন করবেন?**
একটি message queue সার্ভিসগুলোকে সময়, স্থান এবং গতির দিক থেকে decouple করে। Producer-এর প্রয়োজন নেই consumer অনলাইনে বা দ্রুত থাকুক, তাই একটি ধীরগতির বা সাময়িকভাবে অনুপলব্ধ ডাউনস্ট্রিম সার্ভিস producer-এর request ব্যর্থ বা আটকে দেয় না। এটা ট্রাফিক স্পাইকও শুষে নেয় এবং প্রতিটি সার্ভিসকে স্বাধীনভাবে স্কেল করতে দেয়।

**২. Kafka এবং RabbitMQ-এর মধ্যে মৌলিক স্থাপত্যগত পার্থক্য কী?**
RabbitMQ একটি প্রথাগত message broker যেখানে message-গুলো exchange-এর মাধ্যমে queue-তে রুট করা হয় এবং consume ও acknowledge হওয়ার পরে মুছে যায়। Kafka একটি distributed, append-only log যেখানে consumption নির্বিশেষে message-গুলো একটি নির্ধারিত সময়ের জন্য সংরক্ষিত থাকে, এবং একাধিক স্বতন্ত্র consumer group প্রত্যেকে নিজেদের offset-এ একই ডেটা পড়তে (এবং replay করতে) পারে।

**৩. RabbitMQ-কে Kafka-র চেয়ে কখন বেছে নেবেন?**
যখন আপনার জটিল routing logic দরকার (যেমন, message type বা priority অনুযায়ী রুট করা), যেখানে প্রতিটি job ঠিক একজন worker দ্বারা তোলা উচিত এমন ক্লাসিক task/job-queue semantics দরকার, কম operational overhead, অথবা উচ্চ-থ্রুপুট event streaming-এর বদলে per-message acknowledgment এবং retry semantics দরকার।

**৪. Kafka-কে RabbitMQ-র চেয়ে কখন বেছে নেবেন?**
যখন আপনার অত্যন্ত উচ্চ throughput event ingestion দরকার, ঐতিহাসিক event-এর টেকসই replay দরকার, একই event stream consume করা একাধিক স্বতন্ত্র টিম/সার্ভিস দরকার, অথবা আপনি একটি event-sourcing/stream-processing pipeline তৈরি করছেন (যেমন, একই event থেকে রিয়েল-টাইম ড্যাশবোর্ড এবং একটি data lake উভয়কেই ফিড করা)।

**৫. আপনি কীভাবে at-least-once বনাম exactly-once delivery নিশ্চিত করেন?**
At-least-once অর্জিত হয় broker একটি acknowledgment না পাওয়া পর্যন্ত message পুনরায় ডেলিভার করার মাধ্যমে — সফল প্রসেসিংয়ের পরে ack নিজেই হারিয়ে গেলে এতে duplicate হওয়ার ঝুঁকি থাকে। Exactly-once-এর জন্য অতিরিক্ত ব্যবস্থা দরকার: idempotent producers (retry-গুলো deduplicate করা), transactional writes যা message এবং consumer-এর offset উভয়কেই atomically কমিট করে, অথবা, আরও ব্যবহারিকভাবে, একটি idempotent consumer-এর সাথে মিলিত at-least-once delivery (যেমন, একটি আপডেট প্রয়োগ করার আগে একটি ইউনিক message ID পরীক্ষা করা) যাতে পুনরায়-প্রসেসিং-এর কোনো পার্শ্বপ্রতিক্রিয়া না থাকে।

**৬. একটি consumer group কী এবং স্কেলিং-এর জন্য এটি কেন গুরুত্বপূর্ণ?**
একটি consumer group হলো consumer instance-দের একটি সেট যারা সমন্বিতভাবে একটি queue বা topic consume করে, কাজটি এমনভাবে ভাগ করে যাতে প্রতিটি message/partition একবারে group-এর মধ্যে শুধুমাত্র একটি instance দ্বারা সামলানো হয়। এটি আপনাকে অনুভূমিকভাবে প্রসেসিং স্কেল করতে দেয় — throughput বাড়াতে আরও consumer instance যোগ করুন (Kafka-তে partition-এর সংখ্যা পর্যন্ত)।

**৭. একটি consumer যদি একটি message পড়ার পরে কিন্তু acknowledge করার আগে ক্র্যাশ করে তাহলে কী হয়?**
Broker acknowledgment পায় না, তাই এটা message-টিকে অপ্রসেসকৃত বলে বিবেচনা করে এবং পুনরায় ডেলিভার করে — হয় group-এর অন্য একজন consumer-কে, অথবা রিকভার হওয়ার পরে একই consumer-কে ফেরত। এটাই at-least-once delivery-র পেছনের প্রক্রিয়া, এবং এই কারণেই consumer-দের নিরাপদে duplicate delivery সামলানোর জন্য ডিজাইন করতে হয়।

**৮. Message queue-এর প্রেক্ষাপটে backpressure ব্যাখ্যা করুন।**
Backpressure ঘটে যখন producer-রা consumer-দের প্রসেস করার চেয়ে দ্রুত গতিতে message তৈরি করে। একটি queue message বাফার করে এই অমিলটি শুষে নেয়, consumer-কে অভিভূত করা বা request ব্যর্থ করার পরিবর্তে, যা consumer-কে নিজের টেকসই গতিতে ক্যাচ আপ করতে দেয়।

**৯. Kafka কেন পুরনো event "replay" করতে সমর্থন করতে পারে অথচ একটি সাধারণ RabbitMQ queue পারে না?**
Kafka message-গুলোকে একটি অপরিবর্তনীয়, ordered log হিসেবে ডিস্কে একটি নির্ধারিত retention window-এর জন্য (সময় বা আকার-ভিত্তিক) সংরক্ষণ করে, এবং consumer-রা read হওয়ার সময় broker মুছে দেওয়ার বদলে নিজেদের read offset ট্র্যাক করে। RabbitMQ queue consumption-ভিত্তিক — একবার একটি message acknowledge হয়ে গেলে, সেটা সরিয়ে ফেলা হয়, তাই আপনি স্পষ্টভাবে অন্য কোথাও সংরক্ষণ না করলে replay করার মতো কিছু বাকি থাকে না।

**১০. দৃশ্যকল্প: আপনি এমন একটি সিস্টেম তৈরি করছেন যেখানে একটি "order placed" event-কে inventory আপডেট ট্রিগার করতে হবে, একটি কনফার্মেশন ইমেইল পাঠাতে হবে, এবং একটি fraud-detection মডেল আপডেট করতে হবে, আর পরবর্তী কোয়ার্টারে একটি নতুন analytics টিম বিদ্যমান কোড স্পর্শ না করে নিজেদের consumer যোগ করতে চাইতে পারে। কোন প্রযুক্তি এখানে বেশি মানানসই এবং কেন?**
এখানে Kafka বেশি মানানসই কারণ আপনার একই event পড়া একাধিক স্বতন্ত্র consumer আছে, এবং আপনি চান পরবর্তীতে producer বা বিদ্যমান consumer সংশোধন না করেই নতুন consumer যোগ করতে। Kafka-র topic/consumer-group মডেল প্রতিটি টিমকে স্বাধীনভাবে সাবস্ক্রাইব করতে দেয় এবং যোগদানের সময় ঐতিহাসিক event-ও replay করতে দেয়।

**১১. Message ordering কী, এবং Kafka ও RabbitMQ প্রত্যেকে এটি কীভাবে সামলায়?**
Ordering মানে message-গুলো যে ক্রমে তৈরি হয়েছিল সেই ক্রমেই প্রসেস হওয়া। Kafka শুধুমাত্র একটি একক partition-এর মধ্যে ordering নিশ্চিত করে (একই key-সহ message-গুলো একই partition-এ যায় এবং ক্রমানুসারে পড়া হয়); partition জুড়ে কোনো গ্লোবাল order নেই। RabbitMQ একটি একক queue-এর মধ্যে FIFO ordering নিশ্চিত করে, ধরে নিয়ে যে একজন একক consumer আছে; একাধিক প্রতিযোগী consumer থাকলে, পুরো queue জুড়ে per-message ordering-ও নিশ্চিত করা যায় না।

**১২. একটি ছোট-স্কেল প্রজেক্টের জন্য Kafka বেছে নেওয়ার আগে কোন operational trade-off বিবেচনা করা উচিত?**
Kafka-র operational জটিলতা বেশি — এতে সাধারণত একটি ক্লাস্টার পরিচালনা করতে হয় (broker, partition, replication, এবং ZooKeeper অথবা KRaft consensus), যা কম-থ্রুপুট ব্যবহারের ক্ষেত্রে অতিরিক্ত হতে পারে। ছোট workload-এর জন্য RabbitMQ-এর মতো একটি সরল broker (অথবা এমনকি একটি managed cloud queue) পরিচালনা করা অনেক সস্তা হতে পারে।
</content>
