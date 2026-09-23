# Functional vs Non-Functional Requirements

**Difficulty:** Beginner
**Prerequisites:** [01 - What is System Design?](../01-what-is-system-design/README.md)

## Learning Objectives

- Functional requirements এবং non-functional requirements-এর সংজ্ঞা দেওয়া এবং দুটোর মধ্যে স্পষ্ট পার্থক্য করা।
- একটি অস্পষ্ট (ambiguous) প্রম্পট থেকে দুই ধরনের requirements বের করার অনুশীলন করা।
- সাধারণ non-functional ক্যাটাগরিগুলো বোঝা: performance, scalability, availability, consistency, security, এবং cost।
- Back-of-the-envelope calculation ব্যবহার করে scale (users, requests per second, storage) অনুমান করতে শেখা।
- Requirements gathering বাদ দিলে কেন খারাপ design এবং ব্যর্থ interview হয়, তা উপলব্ধি করা।

## Script

### Hook / Intro

কল্পনা করুন, একজন interviewer বললেন: "Design Instagram।" যদি আপনার পরবর্তী পদক্ষেপ সরাসরি একটি database এবং একটি server আঁকা হয়, তাহলে আপনি ইতিমধ্যেই একটি ভুল করে ফেলেছেন — এবং এটিই system design interview-তে সবচেয়ে সাধারণ ভুল। কোনো কিছু design করার আগে, আপনাকে জানতে হবে *আসলে কী* তৈরি করছেন এবং সেটা *কতটা ভালোভাবে* কাজ করতে হবে। এই কাজটিই করে requirements gathering, আর এটি দুটি ভাগে বিভক্ত: functional এবং non-functional requirements। চলুন দুটোই ভেঙে দেখি।

### Functional Requirements কী?

Functional requirements বর্ণনা করে **system-টি কী করবে** — একজন user-এর দৃষ্টিকোণ থেকে প্রকৃত features এবং behaviors। এগুলো এই প্রশ্নের উত্তর দেয়: "আমি যদি এই product ব্যবহার করি, তাহলে আমি এটা দিয়ে কী করতে পারি?"

Instagram-এর জন্য, functional requirements-এর মধ্যে থাকতে পারে: users ছবি ও ভিডিও আপলোড করতে পারবে, users অন্য users-কে follow করতে পারবে, users যাদের follow করে তাদের posts-এর একটি feed দেখতে পারবে, users posts-এ like এবং comment করতে পারবে, users অন্য users বা hashtags খুঁজতে (search) পারবে।

লক্ষ্য করুন, এগুলো সবই এমন জিনিস যা একজন product manager একটি feature spec-এ লিখবেন। এগুলো concrete, testable, এবং end user-এর কাছে সরাসরি দৃশ্যমান। কোনো functional requirement অনুপস্থিত বা ভুল হলে, product তার নির্ধারিত কাজটি সরাসরি করতে পারে না।

### Non-Functional Requirements কী?

Non-functional requirements বর্ণনা করে **system তার কাজগুলো কতটা ভালোভাবে করে** — quality attributes এবং constraints, যেগুলোর অধীনে সেই features-কে কাজ করতে হয়। এগুলো নতুন কোনো feature যোগ করে না; বরং সেই features-কে কোন মানদণ্ড পূরণ করতে হবে তা নির্ধারণ করে।

Non-functional requirements-এর সাধারণ ক্যাটাগরিগুলো হলো:

- **Performance/Latency** — একটি response কত দ্রুত ফিরে আসতে হবে? (যেমন, feed 200ms-এর কম সময়ে লোড হওয়া)
- **Scalability** — system-কে কতজন user এবং কত traffic সামলাতে হবে, এখন এবং ভবিষ্যতে?
- **Availability** — system কত শতাংশ সময় চালু এবং reachable থাকতে হবে? (যেমন, 99.9% uptime)
- **Consistency** — data পরিবর্তন হলে, system-এর সব অংশ সেই পরিবর্তন কত দ্রুত দেখতে পাবে?
- **Durability** — একবার data সংরক্ষণ (save) হয়ে গেলে, তা কি কখনো হারিয়ে যেতে পারে?
- **Security** — কোন data-কে সুরক্ষিত রাখতে হবে, এবং কার থেকে?
- **Cost** — infrastructure-এর জন্য বাজেট কত?

Instagram-এর জন্য, non-functional requirements এরকম হতে পারে: system-কে অবশ্যই 500 মিলিয়ন daily active users সাপোর্ট করতে হবে, feed loads অবশ্যই 300 মিলিসেকেন্ডের কম সময়ে render করতে হবে, platform-কে 99.99% সময় available থাকতে হবে, একবার আপলোড নিশ্চিত (confirmed) হওয়া ছবিগুলো কখনোই হারানো যাবে না, এবং read-to-write ratio ভীষণভাবে reads-এর দিকে ঝুঁকে থাকে (মানুষ যতগুলো post আপলোড করে তার চেয়ে অনেক বেশি post দেখে)।

### এই পার্থক্যটি কেন গুরুত্বপূর্ণ

এখানে মূল বিষয়টি হলো: **functional requirements আপনাকে বলে কী তৈরি করতে হবে; non-functional requirements আপনাকে বলে সেটা কীভাবে তৈরি করতে হবে।** একই functional requirements-ওয়ালা দুটি system — ধরুন, "users ছোট বার্তা (short messages) পোস্ট করতে পারবে" — architecture-এর দিক থেকে সম্পূর্ণ আলাদা হয়ে যেতে পারে, যদি একটিকে 10 জন user সাপোর্ট করতে হয় আর অন্যটিকে 500 মিলিয়ন। Feature-টি একই। কিন্তু engineering একই নয়।

এই কারণেই interviewer-রা ইচ্ছাকৃতভাবে আপনাকে "design Twitter"-এর মতো অস্পষ্ট প্রম্পট দেয়। তারা দেখতে চায়, আপনি ধরে নেওয়ার (assuming) বদলে থেমে clarifying questions জিজ্ঞাসা করবেন কিনা। একজন শক্তিশালী candidate বলবে: "এটা design করার আগে, আমাকে কয়েকটা বিষয় স্পষ্ট করে নিতে দিন — আমরা মোটামুটি কতজন user-কে লক্ষ্য করছি? read versus write ratio কেমন? আমাদের কি strong consistency দরকার, নাকি eventual consistency গ্রহণযোগ্য? এটা কি multi-region সাপোর্ট প্রয়োজন এমন একটি global user base?" এই প্রশ্নগুলো সরাসরি আপনার architecture-কে নির্ধারণ করে — আপনার database-এর পছন্দ, cache দরকার কিনা, একাধিক data center দরকার কিনা, ইত্যাদি।

### Scale কীভাবে অনুমান করবেন (Back-of-the-Envelope)

এখানে একটি কার্যকর দক্ষতা হলো মোটামুটি capacity estimation করা। আপনার নির্ভুল সংখ্যা দরকার নেই — আপনার দরকার order-of-magnitude estimate, যা আপনার design-কে দিকনির্দেশনা দেবে।

ধরুন, আপনাকে বলা হলো app-টিতে 100 মিলিয়ন daily active users আছে, এবং গড়ে প্রতিটি user দিনে 10টি request করে। তার মানে দিনে 1 বিলিয়ন requests। দিনে প্রায় 86,400 সেকেন্ড দিয়ে ভাগ করলে, গড়ে প্রায় সেকেন্ডে 11,600 requests পাওয়া যায়। কিন্তু traffic সমানভাবে বণ্টিত হয় না — peak hours থাকে — তাই peak load অনুমান করতে আপনি এটাকে 2 থেকে 3 গুণ করতে পারেন, যা peak-এ প্রায় সেকেন্ডে 25,000-35,000 requests-এ গিয়ে দাঁড়ায়। এই একটি হিসাবই আপনাকে বলে দেয়: এটা এমন একটি system নয় যা একটি server-এ চলবে। প্রথম দিন থেকেই আপনার load balancing এবং horizontal scaling দরকার — যেসব concept আমরা আসন্ন videos-এ কভার করব।

একই যুক্তি storage-এর ক্ষেত্রেও প্রযোজ্য: যদি প্রতিদিন 10 মিলিয়ন ছবি আপলোড হয়, এবং প্রতিটি ছবির গড় আকার 2MB হয়, তাহলে প্রতিদিন 20TB নতুন data তৈরি হয়। শুধু এই সংখ্যাটাই আপনাকে বলে দেওয়া উচিত যে আপনি একটিমাত্র database-এ ফাইলগুলো ফেলে রাখতে পারবেন না — আপনার distributed storage এবং সম্ভবত একটি CDN দরকার হবে, যা দুটোই আমরা Module 4-এ কভার করি।

### বাস্তব-জগতের উদাহরণ

চলুন সংক্ষেপে দুটি বাস্তব system তুলনা করি। bit.ly-এর মতো একটি **URL shortener**-এর functional requirements হলো "একটি URL সংক্ষিপ্ত করা" এবং "একটি short URL-কে মূল URL-এ redirect করা।" এর non-functional requirements অত্যন্ত কম read latency-এর ওপর জোর দেয় (redirects প্রায় তাৎক্ষণিক হতে হবে) এবং একটি বিশাল read-to-write ratio, কারণ একটি URL একবার তৈরি হয় কিন্তু হাজার হাজার বার click হয়। এটাকে একটি **banking system**-এর সাথে তুলনা করুন, যেখানে "একাউন্টগুলোর মধ্যে টাকা transfer করা" functional requirement-এর সাথে strict consistency এবং durability-এর একটি non-functional requirement আসে — একটি transaction হারানো বা দুইবার গণনা করা একেবারেই চলবে না, এমনকি এর জন্য কিছুটা গতি ত্যাগ করতে হলেও। একই সাধারণ কাঠামো (client, server, database) — কিন্তু সম্পূর্ণ আলাদা priority, কারণ non-functional requirements ভিন্ন।

### Recap

সংক্ষেপে বললে: functional requirements নির্ধারণ করে system *কী* করে — দৃশ্যমান features। Non-functional requirements নির্ধারণ করে এগুলো *কতটা ভালোভাবে* করতে হবে — performance, scale, availability, consistency, security, cost। কোনো architecture প্রস্তাব করার আগে সবসময় দুই ধরনেরই requirements সংগ্রহ করুন, এবং অস্পষ্ট scale বর্ণনা ("অনেক users") কে design-চালিত concrete সংখ্যায় রূপান্তর করতে মোটামুটি back-of-the-envelope math ব্যবহার করুন।

### পরবর্তী কী

এখন যেহেতু আমরা জানি *কী* তৈরি করতে হবে এবং সেটা *কতটা ভালোভাবে* কাজ করতে হবে তা কীভাবে বের করতে হয়, পরবর্তী video-টি একধাপ পিছিয়ে গিয়ে সব কিছুর নিচের plumbing দেখবে: client-server architecture এবং internet আসলে কীভাবে কাজ করে — DNS, IP addresses, TCP/IP, এবং HTTP। আপনার system-এর প্রতিটি request এই pipeline দিয়ে যায়, তাই load balancers, APIs, বা databases নিয়ে কথা বলার আগে এটা বোঝা অপরিহার্য। সেখানে দেখা হচ্ছে।

## Key Takeaways

- Functional requirements = system কী করে (features, user-visible behavior)।
- Non-functional requirements = system এটা কতটা ভালোভাবে করে (performance, scalability, availability, consistency, durability, security, cost)।
- Design করার আগে সবসময় দুই ধরনের requirements-ই স্পষ্ট করুন — এটাই যেকোনো প্রকৃত design process বা interview-এর প্রথম পদক্ষেপ।
- Back-of-the-envelope estimation (requests/second, storage/day) অস্পষ্ট scale-কে এমন সংখ্যায় রূপান্তর করে যা সরাসরি architecture সিদ্ধান্তকে প্রভাবিত করে।
- একই functional requirements, non-functional constraints-এর ওপর নির্ভর করে সম্পূর্ণ ভিন্ন architecture-এর দিকে নিয়ে যেতে পারে।
</content>
