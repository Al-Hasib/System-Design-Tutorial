# System Design কী? Roadmap ও কীভাবে এটি শিখবেন

Prerequisites: None

## Learning Objectives

- সহজ ভাষায় "system design" সংজ্ঞায়িত করা এবং coding/algorithms থেকে এটিকে আলাদা করে চেনা।
- interview এবং real-world engineering career—দুই ক্ষেত্রেই system design কেন গুরুত্বপূর্ণ তা বোঝা।
- আধুনিক systems-এর প্রধান building blocks (clients, servers, databases, caches, queues, load balancers) সম্পর্কে উচ্চ-স্তরের ধারণা রাখা।
- এই course কোন roadmap অনুসরণ করে এবং কীভাবে effectively পড়াশোনা করতে হবে তা জানা।
- intermediate topics-এ যাওয়ার আগে "beginner" system design knowledge কতটা গভীর হওয়া দরকার সে বিষয়ে বাস্তবসম্মত প্রত্যাশা তৈরি করা।

## Script

### Hook / Intro

সবাইকে স্বাগতম, এই System Design সিরিজের একদম প্রথম video-তে। আপনি যদি কখনো কোনো system design interview guide খুলে "sharding," "consistent hashing," বা "CAP theorem"-এর মতো শব্দ দেখে সাথে সাথে অভিভূত হয়ে গিয়ে থাকেন, চিন্তা করবেন না — ঠিক এই কারণেই এই course-টি তৈরি করা হয়েছে। আমরা একদম শূন্য থেকে শুরু করব এবং ধাপে ধাপে এগিয়ে যাব, যতক্ষণ না এই শব্দগুলো আপনার কাছে programming-এর "variable" বা "loop"-এর মতোই স্বাভাবিক মনে হয়।

এই video-তে আমরা তিনটি প্রশ্নের উত্তর দেব: system design আসলে *কী*? এটা নিয়ে কেন মাথা ঘামানো উচিত? আর এই course-টি কীভাবে সাজানো হয়েছে যাতে আপনি সর্বোচ্চ উপকার পেতে পারেন?

### System Design কী?

চলুন এমন একটি সংজ্ঞা দিয়ে শুরু করি যা আপনি সত্যিই কাজে লাগাতে পারবেন। System design হলো একটি নির্দিষ্ট সেট requirements — যেমন কতজন user সাপোর্ট করতে হবে, কত দ্রুত response দিতে হবে, এবং কতটা reliable হতে হবে — পূরণ করার জন্য একটি software system-এর architecture, components, modules, interfaces এবং data flow নির্ধারণ করার প্রক্রিয়া।

লক্ষ্য করুন, এই সংজ্ঞায় কী নেই: কোনো নির্দিষ্ট programming language, নির্দিষ্ট framework, বা নির্দিষ্ট কোনো লাইনের code-এর উল্লেখ নেই। এটাই মূল mental shift। আপনি যখন code লেখেন, তখন সাধারণত আপনি সমাধান করছেন "আমি কীভাবে এই একটি function বা feature সঠিকভাবে implement করব।" আর যখন আপনি system design করেন, তখন আপনি একটু পিছিয়ে গিয়ে প্রশ্ন করছেন "এই application-এর সব অংশ কীভাবে একসাথে খাপ খায়, এবং real-world পরিস্থিতিতে সেই বিন্যাস টিকে থাকবে কি?"

এটাকে bricklayer আর architect-এর পার্থক্যের মতো ভাবুন। একজন bricklayer একটি brick নিখুঁতভাবে বসাতে দক্ষ। একজন architect ঠিক করেন দেয়াল কোথায় বসবে, ভবনটি বাতাস ও ভূমিকম্প কীভাবে সামলাবে, মানুষ করিডোর দিয়ে কীভাবে চলাচল করবে, এবং plumbing ও electrical systems একে অপরের সাথে সংঘর্ষ ছাড়াই কীভাবে সংযুক্ত হবে। দুটো দক্ষতাই গুরুত্বপূর্ণ — কিন্তু system design থাকে architect-এর স্তরে।

### System Design কেন গুরুত্বপূর্ণ?

এটি শেখার পেছনে দুটি বড় কারণ আছে, এবং তারা একে অপরকে শক্তিশালী করে।

প্রথমত, interview-এর কারণ। আপনি যদি প্রায় যেকোনো tech company-তে mid-level বা senior software engineering role-এর জন্য interview দেন, আপনাকে খুব সম্ভবত একটি system design interview মোকাবেলা করতে হবে। coding interview-এর মতো, যেখানে প্রায়শই একটি "সঠিক" optimal সমাধান থাকে, তার বিপরীতে একটি system design interview open-ended। interviewer দেখতে চান আপনি কীভাবে চিন্তা করেন: আপনি কীভাবে requirements স্পষ্ট করেন, কীভাবে trade-offs তৈরি করেন, এবং কীভাবে আপনার reasoning communicate করেন। এখানে খুব কমই একটি সঠিক উত্তর থাকে — বরং প্রদত্ত constraints অনুযায়ী ভালো ও খারাপ trade-offs থাকে।

দ্বিতীয়ত, এবং সততার সাথে বলতে গেলে দীর্ঘমেয়াদে আরও গুরুত্বপূর্ণ, সেটা হলো real-world engineering-এর কারণ। আপনার career যত বাড়বে, আপনাকে শুধু একটি ticket implement করতে বলা হবে না, বরং একটি feature বা পুরো product *কীভাবে* তৈরি করা উচিত তা ঠিক করতে বলা হবে। এটা কি একটি service হওয়া উচিত নাকি তিনটি? আমাদের কি relational database নাকি NoSQL store ব্যবহার করা উচিত? আমাদের কি একটি cache দরকার? আজকের চেয়ে ১০০ গুণ বেশি traffic এলে এই feature-এর কী হবে? এগুলো system design-এর প্রশ্ন, এবং এগুলোর ভালো উত্তর দেওয়াই একজন junior engineer-কে একজন staff বা principal engineer থেকে আলাদা করে।

### আপনি যে Building Blocks শিখবেন

এই course জুড়ে, আমরা ধাপে ধাপে সেই vocabulary এবং components পরিচয় করিয়ে দেব যা আপনি ব্যবহার করেছেন এমন প্রায় প্রতিটি large-scale system-এর ভিত্তি তৈরি করে — যেমন Twitter, Netflix, Uber, বা Amazon। উচ্চ-স্তরে, এই systems তৈরি হয়:

- **Clients** — browsers, mobile apps, বা অন্য যেকোনো service যা কিছু request করে।
- **Servers** — যেসব machine সেই requests process করে এবং responses ফেরত পাঠায়।
- **Databases** — যেখানে data স্থায়ীভাবে সংরক্ষিত, structured করা এবং query করা হয়।
- **Caches** — দ্রুত, সাময়িক storage যা database-এর উপর load কমায় এবং responses দ্রুত করে।
- **Load balancers** — traffic directors যা একাধিক server জুড়ে requests ছড়িয়ে দেয়।
- **Message queues** — buffers যা system-এর বিভিন্ন অংশকে একে অপরের সাথে tightly coupled না হয়ে বা একই সময়ে সরাসরি available না থেকেও যোগাযোগ করতে দেয়।

এই মুহূর্তে, এগুলো হয়তো শুধুই কিছু নাম মনে হতে পারে। এটা সম্পূর্ণ স্বাভাবিক। এই course শেষে, এদের প্রতিটি ঠিক কোন সমস্যার সমাধান করে এবং কখন এটি ব্যবহার করতে হয় তা আপনি স্পষ্টভাবে বুঝবেন।

### এই Course কীভাবে সাজানো হয়েছে

এই course বারোটি module-এ সাজানো হয়েছে, foundations থেকে শুরু করে সম্পূর্ণ interview-style case studies পর্যন্ত:

আমরা শুরু করছি এখানে, **Module 1: Foundations**-এ, যেখানে requirements gathering, internet আসলে কীভাবে কাজ করে, এবং scalability ও reliability-এর মূল ধারণাগুলো covered হবে। এরপর **Module 2** networking এবং HTTP, load balancing, ও APIs-এর মতো communication patterns covered করে। **Module 3** databases এবং storage-এ গভীরে যায় — SQL বনাম NoSQL, indexing, replication, sharding। **Module 4** caching এবং content delivery covered করে। **Module 5** asynchronous systems পরিচয় করিয়ে দেয়: message queues এবং event-driven architecture। **Module 6** consistent hashing, rate limiting, ও consensus algorithms-এর মতো গভীরতর distributed systems concepts-এ যায়। **Module 7** microservices বনাম monoliths-এর মতো architecture patterns covered করে। **Module 8** transport protocols, web server internals, message formats, এবং এতক্ষণ যা covered হয়েছে তার নিচে থাকা security fundamentals-এর একটি স্তর নিচে নামে। **Module 9** database transaction internals এবং storage engines নিয়ে আরও গভীরে যায়, সাথে API alternative হিসেবে GraphQL। **Module 10** distributed coordination এবং scale techniques covered করে — distributed locking, logical clocks, এবং probabilistic data structures। **Module 11** একটি system প্রকৃতপক্ষে production-এ চালাতে যা লাগে তা covered করে — observability, containers, safe deployments, resilience testing, এবং multi-region disaster recovery। এবং সবশেষে, **Module 12** সবকিছুকে একত্রিত করে সম্পূর্ণ, বাস্তবসম্মত case studies দিয়ে — একটি URL shortener, একটি chat app, একটি news feed, এবং আরও অনেক কিছু design করে।

### Real-World Example

চলুন একটি দ্রুত উদাহরণ দিয়ে এটাকে বাস্তবে নিয়ে আসি। কল্পনা করুন আপনাকে "একটি URL shortener" design করতে বলা হয়েছে, যেমন bit.ly। একটি pure coding approach হয়তো সরাসরি একটি hash function লেখা শুরু করবে। কিন্তু একটি system design approach প্রথমে জিজ্ঞাসা করে: আমাদের প্রতিদিন কতগুলো URL shorten করতে হবে? প্রতি সেকেন্ডে redirects (reads) বনাম নতুন URLs (writes)-এর সংখ্যা কত — কারণ এই ratio আমাদের architecture-কে নাটকীয়ভাবে পরিবর্তন করে? পুরনো URL কি কখনো expire হয়? click counts-এর উপর কি আমাদের analytics দরকার? শুধুমাত্র এই প্রশ্নগুলোর উত্তর দেওয়ার পরেই আপনি আপনার database, caching strategy, এবং service architecture বেছে নেওয়া শুরু করেন। সব building blocks হাতে পাওয়ার পর, আমরা Module 12-এ একসাথে এই ঠিক এই system-টিই design করব।

### Recap

চলুন recap করি। System design হলো একটি software system-এর অংশগুলো কীভাবে scale, speed, এবং reliability সম্পর্কিত requirements পূরণের জন্য একসাথে খাপ খায় তা design করা — এটা architect-level thinking, bricklayer-level thinking নয়। এটা গুরুত্বপূর্ণ দুটি কারণে: এটি technical interviews-এর একটি standard অংশ, এবং একজন engineer হিসেবে বেড়ে ওঠার জন্য এটি একটি মূল দক্ষতা। এই course বারোটি module জুড়ে এগিয়ে যায়, foundations দিয়ে শুরু করে সম্পূর্ণ case studies দিয়ে শেষ হয়, এবং প্রতিটি module আগের module-এর vocabulary-এর উপর ভিত্তি করে গড়ে ওঠে।

### পরবর্তীতে যা আসছে

পরের video-তে, আমরা যেকোনো প্রকৃত design process-এর একদম প্রথম practical পদক্ষেপ নেব: requirements কীভাবে সংগ্রহ করে **functional** এবং **non-functional** requirements-এ শ্রেণিবদ্ধ করতে হয় তা শেখা। এটাই সেই ধাপ যা প্রায় সবাই এড়িয়ে যায় যখন তারা সরাসরি whiteboard-এ boxes এবং arrows আঁকা শুরু করে দেয় — এবং এই ধাপ এড়ানোই মূল কারণ কেন এত বেশি design প্রশ্নের সামনে ভেঙে পড়ে। সেখানে দেখা হবে।

## Key Takeaways

- System design ফোকাস করে components (clients, servers, databases, caches, queues) কীভাবে একসাথে খাপ খায় তা requirements পূরণ করার জন্য — আলাদা আলাদা লাইনের code লেখার উপর নয়।
- এটা দুটি কারণে গুরুত্বপূর্ণ: বেশিরভাগ tech companies-এর technical interviews, এবং senior/staff roles-এ real-world career growth।
- খুব কমই একটি "সঠিক" design থাকে — শুধুমাত্র constraints অনুযায়ী ভালো বা খারাপ trade-offs থাকে।
- এই course-এ 12টি module আছে, foundational concepts থেকে শুরু করে সম্পূর্ণ system design case studies পর্যন্ত এগিয়ে যায়।
- প্রতিটি পরবর্তী module সরাসরি Module 1-এ introduce করা vocabulary-এর উপর ভিত্তি করে গড়ে ওঠে।
