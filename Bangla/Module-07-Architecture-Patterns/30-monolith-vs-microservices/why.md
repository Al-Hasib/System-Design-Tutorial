# কেন এই বিষয়টি গুরুত্বপূর্ণ: Monolith vs Microservices

> **এক বাক্যে:** আধুনিক software-এ এটিই সবচেয়ে তাৎপর্যপূর্ণ এবং সবচেয়ে বেশি বিভ্রান্তভাবে নেওয়া architectural সিদ্ধান্ত — team-গুলো technical কারণ দেখিয়ে microservices গ্রহণ করে যখন প্রকৃত justification হয় organizational, এবং এমন একটি সমস্যার জন্য বিশাল জটিলতার খরচ দেয় যা তাদের ছিলই না।

## এই ধারণার আগের জগৎ

**যে monolith তার team-কে ছাড়িয়ে গেছে।** একটি codebase, 400 জন engineer। প্রতিটি deploy-এর জন্য team-গুলোর মধ্যে সমন্বয় দরকার। test suite চলে 90 মিনিট এবং তা অস্থির, তাই merge গুলো সারি বেঁধে থাকে। reporting module-এর একটি memory leak checkout-কে বন্ধ করে দেয়, কারণ এটা একই process। পুরো application-কে একসাথে scale করতে হয়, তাই একটি hot endpoint-এর load সামলাতেই আপনি সবকিছুর 200টি instance চালান। কেউ একটি shared library upgrade করতে পারে না কারণ তা চল্লিশটি অন্য জায়গা ভেঙে দেবে।

**যে microservices migration পরিস্থিতি আরও খারাপ করেছে।** তাই team 60টি service-এ বিভক্ত হয়। এখন একটি একক user action আটটি service অতিক্রম করে। একটি bug মানে আটটি repository পড়া এবং আটটি log-এর সেট সমন্বয় করা। Local development-এর জন্য এক ডজন container চালানো দরকার। Service-গুলো তবুও একটি database share করে, তাই তারা এখনও স্বাধীনভাবে deploy করতে পারে না — আপনি একটি distributed monolith তৈরি করেছেন, যাতে distribution-এর সব জটিলতা আছে কিন্তু autonomy-র কিছুই নেই।

দুটোই বাস্তব, এবং দুটোই সাধারণ। ব্যর্থতার প্যাটার্ন ভুল বেছে নেওয়া নয়; এটা হলো কোন সমস্যা সমাধান করছেন তা না জেনে বেছে নেওয়া।

## এটি যেসব সমস্যা সমাধান করে

### ১. Deployment coupling যা প্রতিটি team-কে ধীর করে দেয়
**আপনি যা দেখেন:** একটি এক-লাইনের পরিবর্তন ship করতে গেলে একটি release train, একটি 90-মিনিটের pipeline, এবং বাকি সবার পরিবর্তন green হওয়ার জন্য অপেক্ষা করতে হয়।

**কেন এটা ঘটে:** একটি deployable unit মানে একটি deployment cadence, যা batch-এর সবচেয়ে ধীর এবং সবচেয়ে ঝুঁকিপূর্ণ পরিবর্তন দ্বারা নির্ধারিত।

**Microservices কীভাবে এটি সমাধান করে:** প্রতিটি service নিজস্ব সময়সূচি, নিজস্ব pipeline এবং নিজস্ব ঝুঁকি নিয়ে deploy হয়। একটি team কাউকে জিজ্ঞেস না করে দিনে দশবার ship করতে পারে। **microservices গ্রহণের এটাই প্রাথমিক, এবং সম্ভবত একমাত্র বাধ্যতামূলক-মানের যুক্তি** — এবং লক্ষ্য করুন এটা একটি organizational সুবিধা, technical নয়।

### ২. একটি জিনিস scale করতে গিয়ে সবকিছু scale করা
**আপনি যা দেখেন:** image-processing endpoint-এর 64 GB RAM দরকার, তাই পুরো application-এর প্রতিটি instance 64 GB পায়, এমনকি যেগুলো শুধু static JSON serve করে সেগুলোও।

**কেন এটা ঘটে:** একটি monolith একটি একক ইউনিট হিসেবে scale হয়। Resource প্রয়োজনীয়তা হলো এর সব function জুড়ে সর্বোচ্চটি।

**Microservices কীভাবে এটি সমাধান করে:** image service-কে 20টি memory-heavy instance-এ এবং API service-কে 100টি সস্তা instance-এ scale করুন। স্বাধীন resource profile, স্বাধীনভাবে tune করা — বড় scale-এ একটি প্রকৃত এবং পরিমাপযোগ্য খরচ জয়।

### ৩. একটি একক failure-এর blast radius
**আপনি যা দেখেন:** একটি অস্পষ্ট background feature-এর একটি bug heap শেষ করে দেয় এবং পুরো application বন্ধ করে দেয়, revenue-critical path সহ।

**কেন এটা ঘটে:** একটি process, একটি memory space, একটি thread-এর সেট। module-এর মধ্যে কোনো isolation নেই।

**Microservices কীভাবে এটি সমাধান করে:** Process এবং network boundary হলো কঠিন boundary। recommendation service-এর OOM checkout service-এর memory খেতে পারে না। circuit breaker-এর সাথে মিলিয়ে, failure সীমাবদ্ধ থাকে।

### ৪. পুরো codebase জুড়ে technology lock-in
**আপনি যা দেখেন:** একটি machine learning feature যা Python-এ তুচ্ছ হতো, তা Java-তে লিখতে হয় কারণ monolith Java-তে তৈরি।

**কেন এটা ঘটে:** একটি codebase, একটি runtime, একটি dependency tree।

**Microservices কীভাবে এটি সমাধান করে:** প্রতিটি service নিজস্ব stack বেছে নেয়। প্রকৃত সুবিধা — এবং সবচেয়ে বেশি অপব্যবহৃত justification-ও, কারণ polyglot fleet operational surface-কে বিশালভাবে বহুগুণ করে দেয়। বেশিরভাগ organization-এর ইচ্ছাকৃতভাবে নিজেদের দুই বা তিনটি stack-এর মধ্যে সীমাবদ্ধ রাখা উচিত।

## যে দাম আপনাকে দিতে হয়

Microservices হলো *স্থানীয়* সরলতার বিনিময়ে *বৈশ্বিক* জটিলতার একটি লেনদেন, এবং বিলটা বড়:

- **প্রতিটি function call একটি network call হয়ে যায়।** এটি fail করতে পারে, timeout হতে পারে, retry হতে পারে, এবং duplicate হতে পারে। Latency ন্যানোসেকেন্ড থেকে মিলিসেকেন্ডে চলে যায়। এই প্রতিটি call-এরই এখন timeout, retry, circuit breaker, এবং fallback দরকার।
- **Transaction অদৃশ্য হয়ে যায়।** Service জুড়ে বিস্তৃত operation-গুলোর জন্য saga বা 2PC দরকার। যে data আগে join করা হতো তা এখন denormalized এবং eventually consistent।
- **Observability একটি বাধ্যতামূলক infrastructure হয়ে যায়।** Distributed tracing, correlation ID, এবং centralized log ছাড়া আপনি কিছুই debug করতে পারবেন না। এটি অবশ্যই split-এর *আগে* থাকতে হবে, পরে নয়।
- **Operational খরচ বহুগুণ বেড়ে যায়।** 60টি service মানে 60টি pipeline, 60 সেট dashboard এবং alert, 60টি on-call surface, 60টি dependency-upgrade stream।
- **Local development কঠিন হয়ে যায়।** ল্যাপটপে system চালানো অসম্ভবও হয়ে যেতে পারে।
- **Boundary ভুল করা ব্যয়বহুল।** ভুল boundary মানে বকবকে cross-service call এবং এমন পরিবর্তন যার জন্য তিনটি team জুড়ে সমন্বিত deploy দরকার — ঠিক সেই coupling যা এড়াতে আপনি বিভক্ত করেছিলেন, এবার network latency যোগ করে।

**দৃঢ়ভাবে সুপারিশকৃত পথ:** একটি ভালোভাবে modularized monolith দিয়ে শুরু করুন। code-এ অভ্যন্তরীণ boundary জোরদার করুন। একটি service শুধু তখনই বের করুন যখন একটি নির্দিষ্ট, বাস্তব যন্ত্রণা — একটি team যা deploy-এ আটকে আছে, একটি component যার scaling profile ভীষণভাবে ভিন্ন — এটাকে justify করে। এই "modular monolith first" পদ্ধতি আপনাকে খরচের একটি ভগ্নাংশে structural সুবিধার বেশিরভাগ অংশ দেয়, এবং boundary গুলো ব্যয়বহুল করে তোলার আগেই সেগুলো আসলে কোথায় তা শিখতে দেয়।

## কখন এটি প্রয়োজন — এবং কখন প্রয়োজন নেই

| Microservices যখন | Monolith যখন |
|---|---|
| অনেক team একটি shared deploy pipeline দ্বারা আটকে আছে | প্রায় 20-30 জনের কম engineer |
| Component গুলোর সত্যিকারভাবে ভিন্ন scaling চাহিদা আছে | Domain এখনও ভালোভাবে বোঝা যায়নি |
| Critical এবং non-critical-এর মধ্যে fault isolation দরকার | আপনার পরিপক্ব CI/CD এবং observability নেই |
| ভিন্ন অংশগুলোর সত্যিকারভাবে ভিন্ন technology দরকার | Independence-এর চেয়ে development speed বেশি গুরুত্বপূর্ণ |
| আপনার ইতিমধ্যেই শক্তিশালী DevOps পরিপক্কতা আছে | Domain জুড়ে strong consistency প্রয়োজন |

## কেন এটি Interview-এ দেখা যায়

এই প্রশ্নটি knowledge test-এর চেয়ে বেশি একটি judgment test। microservices-এর সুবিধা মুখস্থ বলা কম নম্বর পায়; interviewer-রা অনেক ব্যর্থ migration দেখেছেন। উচ্চ-signal উত্তর শুরু হয় "এটা team-এর আকার এবং organizational কাঠামোর উপর নির্ভর করে, traffic-এর উপর নয়" দিয়ে, ডিফল্ট হিসেবে একটি modular monolith সুপারিশ করে, নির্দিষ্ট trigger-গুলোর নাম বলে যা বিভক্ত করাকে justify করবে, এবং খরচ সম্পর্কে খোলামেলা — distributed transaction, observability প্রয়োজনীয়তা, operational overhead। Conway's Law এবং distributed-monolith anti-pattern উল্লেখ করা দেখায় যে আপনি এটি blog-post স্তরের চেয়ে বেশি গভীরভাবে চিন্তা করেছেন।

## এটি কীভাবে সংযুক্ত

Microservices বেছে নেওয়াই Module 6-এর বেশিরভাগ বিষয়কে ঐচ্ছিকের বদলে বাধ্যতামূলক করে তোলে: **distributed transactions** (topic 28), **idempotency** (topic 29), এবং **circuit breakers** (topic 26) সবই প্রয়োজনীয় হয়ে ওঠে। এটি **service discovery** এবং inter-service **communication** (topic 31), একটি **API gateway** (topic 9), **message queues** (topic 20), এবং **observability** (topic 43)-এর প্রয়োজন তৈরি করে। **Domain-driven design** (topic 32) হলো boundary আঁকার discipline, এবং **containers এবং Kubernetes** (topic 44) হলো ফলাফলটি পরিচালনা করার উপায়।

**পরবর্তী:** [Microservices Communication & Service Discovery](../31-microservices-communication-and-service-discovery/why.md) — আপনি যে service গুলো এইমাত্র তৈরি করলেন সেগুলো কীভাবে একে অপরকে খুঁজে পায় এবং কথা বলে।
</content>
