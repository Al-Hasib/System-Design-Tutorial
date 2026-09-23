# Publish-Subscribe Pattern

**কঠিনতা:** Intermediate

## শেখার লক্ষ্যসমূহ (Learning Objectives)

- publish-subscribe (pub-sub) pattern-এর সংজ্ঞা এবং এটি কীভাবে point-to-point queue থেকে আলাদা তা বোঝা
- topic, publisher, subscriber এবং fan-out ব্যাখ্যা করা
- pub-sub কীভাবে producer এবং অনেক স্বাধীন consumer-এর মধ্যে loose coupling অর্জন করে তা বোঝা
- সাধারণ pub-sub implementation চিহ্নিত করা এবং কখন সেগুলো ব্যবহার করতে হবে তা জানা
- pub-sub যে tradeoff নিয়ে আসে (message ordering, delivery guarantees, discoverability) তা শনাক্ত করা

## স্ক্রিপ্ট (Script)

### শুরুর কথা (Hook/Intro)

একটা YouTube channel-এর কথা ভাবুন — আসলে এটাই। যখন আমি একটা ভিডিও আপলোড করি, তখন আমার কোনো ধারণা থাকে না কে সেটা দেখছে। আমি আপনাকে ব্যক্তিগতভাবে কোনো message পাঠাই না। আমি শুধু ভিডিওটা publish করি, আর যারা subscribe করে রেখেছেন তারা সবাই notification পান, যখনই তারা check করেন, যে device থেকেই করেন না কেন। আমি আপনার ফোন নম্বরের কোনো তালিকা রক্ষণাবেক্ষণ করি না। আমি জানি না আপনি এখনই দেখবেন নাকি পরের সপ্তাহে। সেই সম্পর্কটাই — একজন publisher, অনেক স্বাধীন subscriber, তাদের মধ্যে সরাসরি কোনো coupling নেই — distributed system-এ ঠিক এটাই হলো publish-subscribe pattern। চলুন এটা নিয়ে গভীরে যাই।

### Point-to-Point বনাম Publish-Subscribe

গত ভিডিওতে আমরা message queue নিয়ে কথা বলেছিলাম, এবং বেশিরভাগ উদাহরণই ছিল "point-to-point" — একটা message queue-তে যায়, আর ঠিক একজন consumer (অথবা একটা group-এর মধ্যে একজন consumer) সেটা তুলে নিয়ে process করে। কাজ ভাগ করে দেওয়ার জন্য এটা দারুণ: এই job process করো, এই order পাঠাও, এই ছবিটা resize করো। প্রতিটা message ঠিক একবারই handle হয়।

Publish-subscribe এই মডেলটা উল্টে দেয়। একজন consumer একটা message handle করার বদলে, সম্ভবত অনেক স্বাধীন consumer প্রত্যেকে একই message-এর নিজস্ব একটা copy পায়। Producer-কে — এখন যাকে বলা হয় **publisher** — একটা নির্দিষ্ট worker-এর জন্য নির্দিষ্ট queue-তে message পাঠাতে হয় না। এটা একটা **topic**-এ (কখনো কখনো channel বলা হয়) publish করে, আর সেই topic-এ যারাই আগ্রহ দেখিয়ে register করেছেন, সেই যেকোনো সংখ্যক **subscriber** একে অপরের সম্পর্কে কিছু না জেনেই স্বাধীনভাবে একটা copy পান।

মূল মানসিক পরিবর্তনটা হলো: queue-তে একটা message publish করার মানে হলো "কেউ একজন, এই কাজটা করো।" pub-sub-এ একটা message publish করার মানে হলো "এই ঘটনাটা ঘটেছে — যার আগ্রহ আছে, সাড়া দাও।"

### Pub-Sub-এর গঠন: Fan-Out

এখানকার জাদুর শব্দটা হলো **fan-out**। একটা event ভেতরে ঢোকে, আর সেটা প্রতিটা আগ্রহী subscriber-এর কাছে ডুপ্লিকেট হয়ে বেরিয়ে যায়। যদি তিনটা service একটা "user-signed-up" topic-এ subscribe করে থাকে, তাহলে তিনটাই publish হওয়ার মুহূর্তেই ঠিক সেই event পায় — তাদের মধ্যে কোনো coordination প্রয়োজন হয় না, পরে চতুর্থ একটা subscriber যোগ করতে publisher-এর কোডে কোনো পরিবর্তন করতে হয় না।

এটাই pub-sub-কে loosely coupled system-এর মেরুদণ্ড করে তোলে। publisher-এর একমাত্র দায়িত্ব হলো বলা "এটা ঘটেছে।" কে শুনছে বা তারা এটা দিয়ে কী করছে সে সম্পর্কে এর কোনো ধারণা থাকে না। আপনি পরের মাসে একদম নতুন একটা subscriber যোগ করতে পারেন — ধরুন, একটা নতুন team যারা customer-loyalty feature তৈরি করছে এবং জানতে চায় প্রতিবার যখন একজন user sign up করে — আর publisher-এর কোডে এক ফোঁটাও পরিবর্তন হয় না। বড় organization-এ team autonomy-র জন্য এটা বিশাল লাভ: team-গুলো producing service-এর codebase-এ হাত না দিয়ে, যে event নিয়ে তারা যত্নশীল সেটাতে subscribe করে স্বাধীনভাবে build এবং deploy করতে পারে।

### সাধারণ Implementation-সমূহ

আপনি pub-sub-কে বিভিন্ন রূপে দেখতে পাবেন। **Kafka**, যা আমরা গতবার আলোচনা করেছি, স্বাভাবিকভাবেই pub-sub সমর্থন করে কারণ একাধিক consumer group স্বাধীনভাবে একই topic পড়তে পারে — এটাই এর log model-এ built-in fan-out। **Redis Pub/Sub** একটা লাইটওয়েট, in-memory, fire-and-forget সংস্করণ অফার করে — real-time notification-এর জন্য দারুণ যেখানে আপনার durability দরকার নেই, কারণ message persist করা হয় না; যদি publish করার সময় কোনো subscriber connected না থাকে, তাহলে সে কেবল সেই message miss করে। **Google Cloud Pub/Sub**, **AWS SNS**, এবং **Azure Service Bus Topics** হলো managed cloud service যেগুলো বড় scale-এ pub-sub-এর জন্যই বিশেষভাবে তৈরি, প্রায়ই একটা topic-কে per-subscriber queue-এর সাথে জোড় করে যাতে প্রতিটা subscriber durable, at-least-once delivery পায় এমনকি যদি subscriber সাময়িকভাবে অফলাইনও হয়ে যায়। **MQTT**, IoT-তে সাধারণ, একটা লাইটওয়েট pub-sub protocol যা sensor device-এর মতো low-bandwidth, high-latency network-এর জন্য ডিজাইন করা।

লক্ষ্য করুন প্যাটার্নটা: pub-sub একটা কনসেপচুয়াল মডেল, কোনো একক technology নয়। গুরুত্বপূর্ণ হলো এটা বোঝা যে কখন আপনার "N জন আগ্রহী পক্ষকে জানাও" দরকার আর কখন "M জন worker-এর মধ্যে একজনকে কাজ বণ্টন করো" দরকার।

### জানা দরকার এমন Tradeoff

Pub-sub বিনামূল্যে আসে না। প্রথমত, **ordering** আরও জটিল হয়ে ওঠে — যদি আপনার একাধিক publisher থাকে বা একটা topic partition জুড়ে fan হয়ে যায়, তাহলে সবকিছুর জুড়ে একটা global order নিশ্চিত করা কঠিন বা এমনকি অনির্দিষ্টও হতে পারে, বিশেষ করে fire-and-forget সিস্টেমে। দ্বিতীয়ত, **delivery guarantee ব্যাপকভাবে পরিবর্তিত হয়** implementation ভেদে — Redis Pub/Sub হলো at-most-once এবং কেউ না শুনলে message ফেলে দেয়, যেখানে Kafka এবং cloud pub-sub service persistence-এর মাধ্যমে durable, at-least-once delivery অফার করে। তৃতীয়ত, **discoverability এবং debugging কঠিন হয়ে যায়**: যেহেতু publisher জানে না কে subscribe করেছে, একটা বড় সিস্টেমজুড়ে "এই event fire হলে কী ঘটে" তা ট্রেস করতে ভালো documentation, event schema, এবং observability tooling দরকার হতে পারে — নাহলে আপনি এমন একটা বিস্তৃত, implicit dependency graph-এ শেষ করবেন যা নিয়ে reasoning করা কঠিন। team-গুলো যদি event contract নিয়ে discipline বজায় না রাখে তাহলে একে কখনো কখনো অর্ধেক-মজা করে "distributed monolith" বলা হয়।

### বাস্তব-জগতের উদাহরণ

একটা ride-sharing app-এর কথা ভাবুন যখন একজন driver-এর location update হয়। সেই একটামাত্র "location updated" event-এর হয়তো পৌঁছাতে হবে: rider-এর live map view, একটা ETA-recalculation service, একটা surge-pricing engine, এবং GPS spoofing-এর জন্য নজর রাখা একটা fraud-detection system-এ। একটা সাধারণ point-to-point queue দিয়ে, location service-কে এই চারটা consumer সম্পর্কেই জানতে হবে এবং চারটা আলাদা message পাঠাতে হবে, অথবা আরও খারাপ, চারটা আলাদা API call করতে হবে। pub-sub দিয়ে, location service একটা "location updated" event একটা topic-এ publish করে। চারটা service-ই স্বাধীনভাবে subscribe করে। পরবর্তী quarter-এ, যদি company একটা পঞ্চম feature যোগ করে — ধরুন, বন্ধু ও পরিবারের জন্য একটা "share my live trip" link — সেই team শুধু একই topic-এ subscribe করে। আর কোথাও কোনো পরিবর্তন লাগে না।

### সংক্ষিপ্তসার (Recap)

চলুন সবকিছু একসাথে করি। Publish-subscribe হলো একটা messaging pattern যেখানে একজন publisher একটা topic-এ event broadcast করে, আর যেকোনো সংখ্যক স্বাধীন subscriber প্রত্যেকে নিজের একটা copy পায় — একে বলা হয় fan-out। এটা point-to-point queue থেকে একদম আলাদা, যেখানে একটা message একজন consumer-ই handle করে। বড় সিস্টেমে সত্যিকারের loose coupling এবং স্বাধীন team scaling সম্ভব করে pub-sub, যা Kafka, Redis Pub/Sub, SNS, এবং Google Cloud Pub/Sub-এর মতো টুল দিয়ে বাস্তবায়িত হয় — প্রতিটার durability এবং ordering guarantee আলাদা। tradeoff হলো আপনি "কে শুনছে" সে সম্পর্কে কিছুটা visibility হারান, তাই শক্তিশালী event contract এবং observability অপরিহার্য হয়ে ওঠে।

### পরবর্তী কী (What's Next)

Publish-subscribe একটা building block। পরের ভিডিওতে, আমরা zoom out করে এর উপরে নির্মিত বড় architectural philosophy নিয়ে কথা বলব: event-driven architecture, যেখানে পুরো সিস্টেমকে ডিজাইন করা হয় সরাসরি service-to-service call-এর বদলে event তৈরি করা, তাতে সাড়া দেওয়া, এবং সেগুলো চেইন করার চারপাশে। সেখানে দেখা হবে।

## মূল শিক্ষা (Key Takeaways)

- Pub-sub একটা event অনেক স্বাধীন subscriber-এর কাছে broadcast করে (fan-out), point-to-point queue-এর বিপরীতে যেখানে একটা message একজন consumer-এর কাছে যায়।
- Publisher-দের subscriber সম্পর্কে কোনো ধারণা থাকে না — এই loose coupling স্বাধীন team development এবং নতুন consumer সহজে যোগ করা সম্ভব করে।
- Implementation-গুলো লাইটওয়েট/fire-and-forget (Redis Pub/Sub, MQTT) থেকে শুরু করে durable, at-least-once (Kafka, AWS SNS, Google Cloud Pub/Sub) পর্যন্ত বিস্তৃত।
- Ordering এবং delivery guarantee implementation ভেদে যথেষ্ট পরিবর্তিত হয় — যেকোনো একটার উপর নির্ভর করার আগে সূক্ষ্ম বিবরণ যাচাই করুন।
- Implicit dependency-র একটা ট্রেস-অযোগ্য জালের হাত থেকে বাঁচতে শক্তিশালী event schema/contract এবং observability অপরিহার্য।
</content>
