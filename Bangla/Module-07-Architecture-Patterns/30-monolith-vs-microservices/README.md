# Monolith vs Microservices

**Difficulty:** Intermediate/Advanced

## Learning Objectives

এই ভিডিও শেষে, আপনি নিম্নলিখিত বিষয়গুলো করতে সক্ষম হবেন:

- একটি monolithic architecture এবং একটি microservices architecture ঠিক কী, তা সুনির্দিষ্টভাবে সংজ্ঞায়িত করা।
- এই দুই style-এর মধ্যে প্রকৃত trade-off ব্যাখ্যা করা — শুধু "microservices আধুনিক, monolith সেকেলে" এমন কথা না বলে।
- কোন organizational এবং technical সংকেতগুলো একটি style-কে অন্যটির চেয়ে বেছে নেওয়ার ইঙ্গিত দেয়, তা চিহ্নিত করা।
- মাঝামাঝি সমাধান এবং migration path হিসেবে "modular monolith" এবং "strangler fig" pattern ব্যাখ্যা করা।
- দলগুলো যে সবচেয়ে সাধারণ ভুলটি করে তা এড়ানো: যে সমস্যা microservices সমাধান করে সেটা না থাকা সত্ত্বেও microservices গ্রহণ করা।

## Script

### Hook / Intro

একটা ছোট প্রশ্ন: যদি বলি Amazon, Shopify, এবং Stack Overflow তাদের ব্যবসার বিশাল অংশ monolith-এর উপর চালাত — বা এখনও চালায় — তাহলে কি আপনি অবাক হবেন? গত এক দশক ধরে "microservices"-কে প্রায় engineering পরিপক্কতার একটি ব্যাজের মতো ধরা হয়েছে। কিন্তু সত্যিটা আরও জটিল এবং অনেক বেশি আকর্ষণীয়। আজ আমরা hype-এর আড়াল সরিয়ে monolith বনাম microservices নিয়ে কথা বলব একটি প্রকৃত engineering trade-off হিসেবে, যার দুই দিকেই বাস্তব খরচ আছে, যাতে আপনি যখন একটা design interview-তে থাকবেন — বা কর্মক্ষেত্রে সত্যিকারের এই সিদ্ধান্ত নেবেন — তখন শুধু একটা buzzword আওড়ানোর বদলে যুক্তি দিয়ে চিন্তা করতে পারেন।

### Monolith আসলে কী?

একটি monolithic architecture হলো এমন একটি system যেখানে সব functionality — user service, order service, payment logic, notification logic — একটিমাত্র codebase-এ থাকে এবং একটিমাত্র unit হিসেবে deploy হয়। এর মানে এই নয় যে এটি খারাপভাবে ডিজাইন করা। একটি monolith-এর ভেতরে সম্পূর্ণ পরিচ্ছন্ন module, তাদের মধ্যে সুস্পষ্ট interface, এবং একটি layered architecture থাকতেই পারে। এর নির্ধারক বৈশিষ্ট্য "এলোমেলো কোড" নয়, বরং deployment boundary: একটি build, একটি artifact, একটি process (অথবা একই process-এর একগুচ্ছ অভিন্ন কপি একটি load balancer-এর পেছনে), সাধারণত একটি database।

একটি monolith-কে একটি একক রেস্টুরেন্ট রান্নাঘরের মতো ভাবুন। প্রতিটি স্টেশন — grill, salad, dessert — একই ছাদের নিচে, একজন head chef সবকিছু সমন্বয় করেন, এবং পুরো রান্নাঘর একসঙ্গে খোলে বা বন্ধ হয়। যদি dessert স্টেশনে আগুন লাগে, তাহলে পুরো রান্নাঘরই বন্ধ করতে হতে পারে।

### Microservices আসলে কী?

Microservices architecture সেই একই functionality-কে একগুচ্ছ স্বাধীনভাবে deployable service-এ ভাগ করে দেয়, যেখানে প্রতিটি service তার নিজস্ব data এবং নিজস্ব release cycle-এর মালিক, এবং network-এর মাধ্যমে যোগাযোগ করে — সাধারণত REST, gRPC, বা asynchronous messaging-এর মাধ্যমে, যা আমরা Module 5-এ কভার করেছি। প্রতিটি service এতটাই ছোট যে একটিমাত্র team সম্পূর্ণভাবে সেটির মালিক হতে পারে: এটি বানানো, deploy করা, scale করা, এবং এর জন্য on-call থাকা।

এখন এটা একটি রান্নাঘরের বদলে একটি food court-এর মতো। নুডল স্টল, বার্গার স্টল, এবং স্মুদি স্টল প্রত্যেকেই স্বাধীনভাবে পরিচালিত ব্যবসা। যদি নুডল স্টলে কোনো health inspection সমস্যা হয় এবং সেটা বন্ধ হয়ে যায়, বার্গার স্টল তবুও গ্রাহকদের সেবা দিতে থাকে। কিন্তু এখন আপনার দরকার shared infrastructure — বসার জায়গা, একটি payment system যা সব vendor জুড়ে কাজ করে, তাদের মধ্যে সমন্বয় — যার কোনোটাই তখন ছিল না যখন সবকিছু একটিই রান্নাঘর ছিল। সেই সমন্বয়ই হলো সেই খরচ যা আপনি গ্রহণ করছেন।

### প্রকৃত Trade-off গুলো

একটু স্পষ্ট করে বলা যাক, কারণ এখানেই interview জেতা বা হারা নির্ধারিত হয়।

**Monolith-এর সুবিধা:** development-এর সরলতা — একটিমাত্র codebase clone করে চালানো যায়। deployment-এর সরলতা — একটি CI/CD pipeline, রোলব্যাক করার মতো একটিই জিনিস। strong consistency সহজ, কারণ আপনার সম্ভবত একটিই database আছে এবং আপনি real ACID transaction ব্যবহার করতে পারেন। Cross-cutting পরিবর্তন — যেমন তিনটি module-এ ব্যবহৃত একটি field-এর নাম পরিবর্তন — একটিমাত্র pull request, একাধিক team-এর সমন্বিত rollout নয়। Debug করা সহজ: একটি process, একগুচ্ছ log, HTTP request থেকে database call পর্যন্ত একটিমাত্র stack trace।

**Monolith-এর অসুবিধা:** codebase এবং team বড় হওয়ার সাথে সাথে build time বাড়ে, test suite বড় হয়, এবং merge conflict বহুগুণ বেড়ে যায়। আপনাকে পুরো application scale করতে হয় এমনকি যদি শুধু একটি অংশ — যেমন image processing — bottleneck হয়, যা resource নষ্ট করে। একটি module-এর একটি bug পুরো process-কে crash করাতে পারে। এবং technology choice গোটা system জুড়ে বাঁধা থাকে — আপনি সহজে একটা performance-critical অংশের জন্য Go এবং data science অংশের জন্য Python ব্যবহার করতে পারেন না।

**Microservices-এর সুবিধা:** স্বাধীন deployability — checkout team প্রতিদিন দশবার ship করতে পারে search team-এর সাথে সমন্বয় ছাড়াই। স্বাধীন, লক্ষ্যভিত্তিক scaling — লোড বাড়লে আপনি শুধু image-processing service scale করেন। Fault isolation — যদি recommendation service ভেঙে পড়ে, checkout তবুও কাজ করতে পারে, বিশেষত Module 6-এর circuit breaker pattern-এর সাহায্যে। Team গুলো প্রতিটি কাজের জন্য সঠিক tool বেছে নিতে পারে, এবং বড় organization গুলো অনেকগুলো ছোট, স্বায়ত্তশাসিত team দিয়ে scale করতে পারে — এটি আসলে যতটা না technical, তার চেয়ে বেশি একটি organizational pattern, যাকে প্রায়ই "Conway's Law"-এর উল্টো রূপ বলা হয়: আপনি আপনার system boundary গুলো এমনভাবে ডিজাইন করেন যাতে সেগুলো আপনার কাঙ্ক্ষিত team boundary-র সাথে মেলে।

**Microservices-এর অসুবিধা, এবং এই অংশটাই মানুষ কম গুরুত্ব দেয়:** আপনি in-process function call-কে network call দিয়ে বদলেছেন, যা ধীর এবং নতুন নতুন উপায়ে ব্যর্থ হতে পারে — timeout, partial failure, retry। Distributed transaction সত্যিকার অর্থেই কঠিন হয়ে ওঠে — service জুড়ে আর বিনামূল্যে ACID পাওয়া যায় না, এই কারণেই Module 6-এ saga এবং two-phase commit নিয়ে সময় ব্যয় করা হয়েছিল। আপনার service discovery দরকার হবে, যা আমাদের ঠিক পরের ভিডিওর বিষয়। "এই request-এর 800ms লাগল কেন" এই প্রশ্নের উত্তর দিতেই আপনার distributed tracing এবং centralized logging দরকার। আপনার infrastructure এবং operational জটিলতা — Kubernetes, service mesh, API gateway — বিশালভাবে বেড়ে যায়। এবং ছয়টি service স্পর্শ করে এমন একটি end-to-end flow test করা একটি monolith test করার চেয়ে নাটকীয়ভাবে কঠিন।

### আসলে সিদ্ধান্ত কীভাবে নেবেন?

আমি আপনাকে যে heuristic দেব তা হলো: একটি monolith দিয়ে শুরু করুন, এবং একটি ভালোভাবে modularized monolith-কে প্রাধান্য দিন — যাকে কখনো কখনো "modular monolith" বলা হয় — যেখানে code ইতিমধ্যে সুস্পষ্ট boundary-সহ module-এ ভাগ করা এবং তাদের মধ্যে সরাসরি database coupling নেই, যদিও এটি একটিমাত্র unit হিসেবে deploy হয়। এটি আপনাকে অপারেশনাল খরচ ছাড়াই microservices-এর বেশিরভাগ maintainability সুবিধা দেয়, এবং ভবিষ্যতে বিভক্ত করা অনেক সহজ করে তোলে কারণ seam গুলো ইতিমধ্যেই বিদ্যমান।

আপনি তখনই microservices-এর দিকে এগোন যখন আপনার কাছে সুনির্দিষ্ট সংকেত থাকে: আপনার team এমন আকারে বেড়ে গেছে যা একটি deployable unit ক্রমাগত merge যন্ত্রণা ছাড়া সমর্থন করতে পারে না; নির্দিষ্ট component-গুলোর scaling চাহিদা ভীষণভাবে ভিন্ন — একটি video transcoding service বনাম একটি user-profile service; availability-র কারণে system-এর বিভিন্ন অংশের স্বাধীনভাবে fail করা দরকার; অথবা ভিন্ন team-এর সত্যিকারভাবেই নিজেদের অংশের মালিকানা এবং deploy স্বাধীনভাবে করা প্রয়োজন। "strangler fig" pattern হলো প্রমিত migration path — আপনি monolith-এর সামনে একটি facade বা API gateway বসান এবং ধীরে ধীরে পৃথক পৃথক capability খুলে ফেলে সেগুলোকে নতুন, স্বাধীন service দিয়ে প্রতিস্থাপন করেন, ধাপে ধাপে traffic redirect করে, ঝুঁকিপূর্ণ big-bang rewrite করার বদলে।

### বাস্তব-জগতের উদাহরণ

Segment, customer data platform-টি, এখানে একটি বিখ্যাত case study। তারা একটি monolith হিসেবে শুরু করেছিল, আক্রমণাত্মকভাবে microservices-এ migrate করেছিল — শত শত service — এবং তারপর 2018-এ তারা প্রকাশ্যে লিখেছিল যে তারা সেই service-গুলোর একটা বড় অংশ আবার একটি monolith-এ একত্রিত করছে। কেন? তাদের microservices-গুলোর load এবং failure mode অত্যন্ত পরস্পর সম্পর্কিত ছিল, তাই isolation-এর সুবিধা কখনোই বাস্তবায়িত হয়নি, কিন্তু তারা প্রতিদিন পুরো operational tax দিয়ে যাচ্ছিল — on-call বোঝা, deployment জটিলতা। এটি একটি চমৎকার স্মারক যে microservices একটি নির্দিষ্ট সমস্যাগুচ্ছের জন্য একটি tool, সার্বজনীন upgrade নয়। এর বিপরীতে Amazon বা Netflix-এর কথা ভাবুন, যেখানে বিশাল scale, শত শত স্বাধীন team, এবং ভীষণভাবে ভিন্ন service-level প্রয়োজনীয়তা microservices-কে একটি সুস্পষ্ট জয় করে তোলে। সঠিক উত্তর সম্পূর্ণভাবে আপনার প্রেক্ষাপটের উপর নির্ভর করে।

### সারসংক্ষেপ

চলুন সবকিছু একসাথে বাঁধি। একটি monolith হলো একটিমাত্র deployable unit, সাধারণত একটি shared database সহ; microservices হলো অনেকগুলো স্বাধীনভাবে deployable unit, প্রতিটি নিজস্ব data-র মালিক, network-এর মাধ্যমে কথা বলে। Monolith জেতে সরলতা, consistency, এবং কম operational overhead-এ; microservices জেতে স্বাধীন scaling, fault isolation, এবং team autonomy-তে — distributed system জটিলতার বিনিময়ে। ডিফল্ট হিসেবে একটি modular monolith বেছে নিন, এবং microservices-এ বিভক্ত করুন যখন আপনার কাছে একটি সুনির্দিষ্ট organizational বা scaling কারণ থাকে, শুধু এটা ট্রেন্ডি বলে নয়।

### এরপর কী?

একবার আপনার microservices হয়ে গেলে, তাদের একে অপরের সাথে কথা বলতে হবে এবং runtime-এ একে অপরকে খুঁজে পেতে হবে — একটি service শুধু `localhost` কল করতে পারে না। পরবর্তী ভিডিওতে, আমরা microservices communication pattern এবং service discovery নিয়ে গভীরে যাব: order service আসলে কীভাবে জানে inventory service-এর network address, বিশেষত যখন instance-গুলো ক্রমাগত চালু হচ্ছে, বন্ধ হচ্ছে, এবং একটি cluster-এর মধ্যে ঘুরে বেড়াচ্ছে?

## Key Takeaways

- একটি monolith তার একক deployment unit দ্বারা সংজ্ঞায়িত, code quality দ্বারা নয় — একটি monolith অভ্যন্তরীণভাবে ভালোভাবে modularized হতে পারে।
- Microservices সরলতা এবং সহজ consistency-র বিনিময়ে স্বাধীন scaling, fault isolation, এবং team autonomy পায়।
- Microservices-এর সবচেয়ে বড় লুকানো খরচ হলো operational: service discovery, distributed tracing, network failure handling, এবং service জুড়ে বিনামূল্যে ACID transaction ছেড়ে দেওয়া।
- একটি "modular monolith" একটি শক্তিশালী ডিফল্ট যা strangler fig pattern-এর মাধ্যমে ভবিষ্যতের migration path সংরক্ষণ করে।
- Segment-এর প্রকাশ্য monolith-consolidation গল্প দেখায় যে microservices একটি লক্ষ্যভিত্তিক tool, ডিফল্ট best practice নয়।
</content>
