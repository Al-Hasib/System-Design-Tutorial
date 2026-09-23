# কেন এই বিষয়টি গুরুত্বপূর্ণ: Design a URL Shortener

> **এক বাক্যে:** এটা system design interview-এর "hello world" — এত ছোট যে 40 মিনিটে শেষ করা যায়, আর এত গভীর যে এটা আপনাকে ID generation, চরম read/write skew, এবং একটি cache "ভালো হওয়া" আর একটি cache "load-bearing হওয়া"-র মধ্যে পার্থক্য মোকাবিলা করতে বাধ্য করে।

## এই Case Study কেন আছে

একটি URL shortener শুনতে তুচ্ছ মনে হয়: একটি mapping store করো, lookup-এ redirect করো। এই আপাত সরলতাই ঠিক এই কারণ যে এটা standard opening problem। এখানে লুকানোর মতো কোনো domain জটিলতা নেই, তাই আলোচনা সরাসরি fundamentals-এ চলে যায় — এবং একজন interviewer খুব দ্রুত বুঝতে পারেন একজন candidate design করার আগে estimate করে, নাকি সরাসরি box আঁকা শুরু করে দেয়।

এটাও এমন একটি সমস্যা যেখানে naive উত্তর এবং ভালো উত্তরের মধ্যে ব্যবধান, খরচের তুলনায় সবচেয়ে বড়। "URL-টা hash করো এবং store করো" একটি পাঁচ-সেকেন্ডের উত্তর যা প্রথম follow-up প্রশ্নেই ভেঙে পড়ে।

## যেসব Design সমস্যা এটি আপনাকে সমাধান করতে বাধ্য করে

### ১. Coordination ছাড়াই ছোট, unique ID তৈরি করা
**সমস্যা:** আপনার একটি 6-8 character-এর code দরকার যা কোটি কোটি URL জুড়ে unique, একই সময়ে অনেক server দ্বারা তৈরি, কোনো collision ছাড়া এবং কোনো central bottleneck ছাড়া।

**কেন এটা কঠিন:** একটি auto-incrementing database ID uniqueness দেয় কিন্তু একটি একক sequence দরকার — একটি coordination point এবং একটি scaling limit — এবং এটা code-গুলোকে guessable ও enumerable করে তোলে। লম্বা URL hash করলে collision তৈরি হয় যা detect ও resolve করতে হবে, মানে প্রতিটি write-এর আগে একটি read। একটি random code-এর জন্য একটি uniqueness check দরকার যা keyspace পূর্ণ হতে থাকলে ধীর হতে থাকে।

**আপনি যা শিখবেন:** counter-plus-base62 পদ্ধতি এবং কেন base62 সাত character-এ 62^7 ≈ 3.5 trillion code দেয়। প্রতি-write coordination সরাতে প্রতি server-এ ID range pre-allocate করা (একটি ticket server অথবা একটি Snowflake-style scheme)। এবং security-র ফলাফল — sequential code enumerable, তাই যে কেউ আপনার পুরো database ঘুরে দেখতে পারে — যা একটি non-functional requirement কীভাবে একটি technical পছন্দ বদলে দেয় তার সত্যিকারের ভালো উদাহরণ।

### ২. 100:1 বা তার চেয়েও খারাপ একটি read/write ratio
**সমস্যা:** Write বিরল (একটি link তৈরি করা) এবং read বিশাল (প্রতিটি click, চিরকালের জন্য, প্রায়ই burst-এ যখন একটি link viral হয়)।

**কেন এটা কঠিন:** দুটোকেই optimize করা একটাকে optimize করার চেয়ে একদম ভিন্ন architecture।

**আপনি যা শিখবেন:** পুরো course-এ asymmetric optimization-এর এটাই সবচেয়ে পরিষ্কার উদাহরণ। Read আক্রমণাত্মক caching পায় (জনপ্রিয় link-গুলোর working set খুবই ছোট এবং data immutable, তাই cache invalidation — সাধারণত সবচেয়ে কঠিন অংশ — এখানে প্রায় থাকেই না), read replica, এবং redirect-এর জন্য নিজেই একটি CDN বা edge layer। Write একটি সহজ, সঠিক path পায়। Design করার আগে এই skew চিনে নেওয়া এবং নাম দেওয়াই এই সমস্যায় সবচেয়ে বেশি নম্বর দেওয়া move।

### ৩. Capacity estimation যা সত্যিই design-কে জানায়
**সমস্যা:** পাঁচ বছরের জন্য কত storage? Peak-এ প্রতি সেকেন্ডে কত request? এটা কি একটি database-এ ফিট করে?

**কেন এটা কঠিন:** Candidate-রা হয় এটা বাদ দেয়, নয়তো এমন সংখ্যা হিসাব করে যা পরে তারা উপেক্ষা করে।

**আপনি যা শিখবেন:** estimate করার এবং তারপর ফলাফলটা *ব্যবহার* করার অভ্যাস। প্রতি record কয়েকশ byte গুণ কয়েক বিলিয়ন record মানে কয়েকশ gigabyte — যা একটি machine-এ আরামসে ফিট করে, মানে সৎ উত্তর হলো আপনার হয়তো sharding-ই দরকার নেই। "এটা জটিল হওয়ার দরকার নেই" বলার মতো ইচ্ছা একটা শক্তিশালী সংকেত, এবং estimate-টাই আপনাকে সেটা বলার অধিকার দেয়।

### ৪. যে details একটি সম্পূর্ণ উত্তরকে আলাদা করে
**সমস্যা:** Custom alias, link expiry, click-এ analytics, এবং HTTP redirect status-এর পছন্দ।

**আপনি যা শিখবেন:** Custom alias-এর জন্য একটি uniqueness check এবং একটি reserved-words list দরকার। Expiry-র জন্য একটি TTL এবং একটি cleanup strategy দরকার (lazy deletion অথবা একটি background job)। Analytics synchronous redirect path-এ থাকা উচিত নয় — একটি queue-তে একটি click event publish করুন এবং asynchronously process করুন, নাহলে আপনি আপনার সবচেয়ে দ্রুত operation-কে আপনার সবচেয়ে ধীর operation-এর সাথে couple করে ফেলেছেন। এবং 301 বনাম 302 একটা প্রকৃত trade-off: একটি permanent redirect browser দ্বারা cache হয় এবং তাই দ্রুত ও সস্তা, কিন্তু আপনি সেই click-গুলো আর কখনো দেখবেন না, তাই analytics গুরুত্বপূর্ণ হলে আপনি 302 ব্যবহার করবেন।

## ভুল করলে যা খরচ হয়

- **Estimation বাদ দেওয়া** এমন একটি জিনিসের জন্য একটি sharded, multi-region system design করায় যা একটি node-এই ফিট করে যায় — এবং interviewer-রা এটাকে right-size করতে না পারার অক্ষমতা হিসেবে পড়েন।
- **URL hash করা** collision address না করে, এটাই সবচেয়ে সাধারণ অসম্পূর্ণ উত্তর।
- **Redirect path-এ analytics রাখা** একটি sub-10 ms operation-কে এমন একটিতে পরিণত করে যা আপনার analytics pipeline-এর availability-র উপর নির্ভরশীল।
- **Enumeration উপেক্ষা করা** মানে এমন একটি design যা কেউ যত link কখনো তৈরি করেছে তার সবকিছুই leak করে।
- **Over-engineering** — একটি key-value lookup-এর জন্য Kafka, microservices, এবং একটি custom consensus layer প্রস্তাব করা — under-engineering-এর চেয়ে বেশি কঠোরভাবে বিচার করা হয়, কারণ এটা reasoning-এর বদলে reflex বোঝায়।

## কেন Interviewer-রা এটাই বেছে নেন

এটা industry-র সবচেয়ে common warm-up, এবং এটা একটা calibration tool। কয়েক মিনিটের মধ্যেই interviewer জেনে যান আপনি প্রথমে requirements স্পষ্ট করেন কিনা, আপনি estimate করেন কিনা, আপনি dominant access pattern চিহ্নিত করতে পারেন কিনা, এবং আপনি জটিলতা যোগ করা থেকে বিরত থাকতে পারেন কিনা। এই প্রতিটি আচরণই পরের কঠিনতর সমস্যাগুলোতে আরও বেশি গুরুত্বপূর্ণ, যে কারণে এটা প্রথমে আসে।

## এটি কীভাবে সংযুক্ত

এই case study **requirements এবং estimation** (topic 2), pure key-value workload-এর জন্য **SQL vs NoSQL** (topic 11), **indexing** (topic 12), load-bearing component হিসেবে **caching** (topic 17), **CDN** edge redirect (topic 18), scale দরকার হলে **sharding** এবং **consistent hashing** (topic 14, 24), link creation-এ **rate limiting** (topic 25), দ্রুত "এই code কি নেওয়া হয়ে গেছে?" check-এর জন্য **Bloom filter** (topic 42), এবং click analytics-এর জন্য asynchronous **queue** (topic 20) অনুশীলন করায়।

**পরবর্তী:** [Design a Rate Limiter](../49-design-a-rate-limiter/why.md) — একটি ছোট সমস্যা যার পুরো কঠিনতা distributed details-এর মধ্যে লুকিয়ে আছে।
