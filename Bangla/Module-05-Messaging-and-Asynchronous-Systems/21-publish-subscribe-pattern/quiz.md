# অনুশীলন ও ইন্টারভিউ প্রশ্ন (Practice & Interview Questions)

**১. একটা point-to-point queue এবং publish-subscribe-এর মধ্যে মূল পার্থক্য কী?**
একটা point-to-point queue-তে, প্রতিটা message ঠিক একজন consumer (অথবা একটা consumer group-এর মধ্যে একজন) consume করে। publish-subscribe-এ, প্রতিটা message একটা topic-এর প্রতিটা subscriber-এর কাছে স্বাধীনভাবে পৌঁছায় — এই duplication-কেই বলা হয় fan-out।

**২. সরাসরি service-to-service call-এর তুলনায় pub-sub কেন coupling কমায়?**
Publisher-এর শুধু জানতে হয় সে কোন topic-এ publish করছে, কোনো subscriber-এর পরিচয়, সংখ্যা, বা আচরণ সম্পর্কে নয়। Publisher-এর কোডে কোনো পরিবর্তন ছাড়াই subscriber যোগ বা অপসারণ করা যায়, যা সিস্টেমগুলোকে decouple করে এবং স্বাধীন development ও deployment সম্ভব করে।

**৩. একটা pub-sub use case-এ Kafka-র বদলে Redis Pub/Sub কখন বেছে নেবেন তার একটা উদাহরণ দিন।**
Redis Pub/Sub লাইটওয়েট, real-time, low-stakes notification-এর জন্য (যেমন, একটা collaborative editor-এ live cursor position, অথবা ক্ষণস্থায়ী chat "typing" indicator) ভালো ফিট, যেখানে মাঝেমধ্যে একটা message হারিয়ে যাওয়া গ্রহণযোগ্য এবং আপনার durability বা replay দরকার নেই। এখানে Kafka তার operational জটিলতা এবং persistence overhead-এর কারণে অতিরিক্ত।

**৪. Redis Pub/Sub-এ যদি বর্তমানে কোনো subscriber connected না থাকে তাহলে একটা message-এর কী হয়?**
এটা হারিয়ে যায়। Redis Pub/Sub হলো fire-and-forget এবং at-most-once — এটা message persist করে না, তাই publish করার সময় যে subscriber সক্রিয়ভাবে connected নেই সে সেই message কখনোই পায় না।

**৫. Redis Pub/Sub-এর বিপরীতে AWS SNS এবং Google Cloud Pub/Sub সাধারণত কীভাবে durable, at-least-once delivery অর্জন করে?**
তারা publish করা message persist করে এবং, অনেক setup-এ, প্রতিটা subscriber-কে তার নিজস্ব durable queue-এর সাথে জোড় করে (যেমন, SNS সাধারণত per-subscriber SQS queue-তে fan out করে) যাতে subscriber সাময়িকভাবে অফলাইন থাকলেও, message deliver ও acknowledge না হওয়া পর্যন্ত অপেক্ষা করে, প্রয়োজনে retry করে।

**৬. একটা pub-sub সিস্টেমে global message ordering guarantee করা কেন কঠিন হতে পারে?**
যখন একাধিক publisher, একাধিক partition, বা একাধিক subscriber সমান্তরালে process করে, তখন পুরো সিস্টেমজুড়ে event-এর একটা একক, সার্বজনীনভাবে সম্মত ক্রম থাকে না — প্রতিটা subscriber ভিন্ন আপেক্ষিক সময়ে message দেখতে পারে, এবং কিছু implementation একটা fanned-out topic জুড়ে কোনো ordering-ই guarantee করে না।

**৭. Pub-sub কী operational/organizational ঝুঁকি নিয়ে আসে, এবং team-রা এটা কীভাবে প্রশমিত করে?**
যেহেতু publisher জানে না তাদের event কে consume করে, তাই একটা event fire হলে কী ঘটে তা ট্রেস করা কঠিন হয়ে ওঠে, যা একটা implicit এবং সম্ভাব্যভাবে বিস্তৃত dependency graph-এর দিকে নিয়ে যায় (কখনো কখনো "distributed monolith" বলা হয়)। Team-রা এটা প্রশমিত করে ভালোভাবে documented event schema/contract, event catalog, এবং distributed tracing/observability tooling দিয়ে।

**৮. পরিস্থিতি: একটা social media platform-এর একজন user যখন post করেন তখন তার follower-দের জানানো দরকার। সময়ের সাথে বিভিন্ন team নতুন feature (push notification, activity feed generation, trending-topic detection) যোগ করে। আপনি এটা pub-sub দিয়ে কীভাবে ডিজাইন করবেন, এবং কেন point-to-point নয়?**
একজন user post করলে একটা "post-created" event একটা topic-এ publish করুন। প্রতিটা feature team (push notification, feed generation, trending detection) স্বাধীনভাবে subscribe করে। একটা point-to-point queue posting service-কে প্রতিটা downstream feature সম্পর্কে জানতে এবং সরাসরি message পাঠাতে বাধ্য করবে, প্রতিবার একটা নতুন feature/team যোগ হলে producer-এ কোড পরিবর্তন প্রয়োজন হবে — pub-sub এই coupling সম্পূর্ণভাবে এড়িয়ে যায়।

**৯. Kafka consumer group এবং pub-sub pattern-এর মধ্যে সম্পর্ক কী?**
Kafka-তে প্রতিটা consumer group একটা topic-এর স্বাধীন subscriber-এর মতো কাজ করে — একটা group-এর মধ্যে, কাজ partition জুড়ে load-balance করা হয় (point-to-point style-এ), কিন্তু ভিন্ন ভিন্ন group জুড়ে, একই message প্রতিটা group-এ স্বাধীনভাবে deliver হয়, যা ঠিক pub-sub-এর fan-out আচরণ। Consumer group কীভাবে configure করা হয়েছে তার উপর নির্ভর করে Kafka কার্যকরভাবে দুটো pattern-ই সমর্থন করে।

**১০. MQTT কী এবং IoT-এর প্রেক্ষাপটে এটা pub-sub-এর সাথে কেন সাধারণত ব্যবহৃত হয়?**
MQTT একটা লাইটওয়েট publish-subscribe messaging protocol যা সীমাবদ্ধ device এবং low-bandwidth, high-latency, বা অনির্ভরযোগ্য network-এর জন্য ডিজাইন করা। এর ছোট message overhead এবং configurable Quality of Service (QoS) level একে sensor-এর telemetry data একটা broker-এ publish করার মতো পরিস্থিতিতে উপযুক্ত করে তোলে, যা এটা একাধিক monitoring বা automation subscriber-এ fan out করে।

**১১. একটা financial transaction event-এর জন্য একটা fire-and-forget pub-sub সিস্টেম কেন অনুপযুক্ত হতে পারে, এবং তার বদলে আপনি কী ব্যবহার করবেন?**
Financial transaction event সাধারণত হারানো উচিত নয় — একটা হারিয়ে যাওয়া "payment completed" event মানে হতে পারে একজন customer-এর কাছ থেকে টাকা কেটে নেওয়া হয়েছে কিন্তু downstream system কখনো তা রেকর্ড করেনি। আপনি একটা durable, at-least-once pub-sub implementation (যেমন, Kafka অথবা SNS/SQS) idempotent consumer-এর সাথে ব্যবহার করবেন, plain Redis Pub/Sub-এর মতো একটা fire-and-forget সিস্টেমের বদলে।

**১২. বিদ্যমান subscriber-দের কোনো ঝুঁকি ছাড়াই একটা বিদ্যমান pub-sub topic-এ একদম নতুন একটা consumer কীভাবে যোগ করবেন?**
কেবল নতুন service-টাকে বিদ্যমান topic-এ subscribe করান (অথবা, Kafka-র ক্ষেত্রে, topic-এর শুরু থেকে বা বর্তমান offset থেকে পড়ার জন্য একটা নতুন consumer group তৈরি করুন)। যেহেতু subscriber-রা স্বাধীন, এর জন্য publisher বা অন্য কোনো subscriber-এ কোনো পরিবর্তনের প্রয়োজন হয় না, এবং তাদের processing ব্যাহত হওয়ার কোনো ঝুঁকি থাকে না।
</content>
