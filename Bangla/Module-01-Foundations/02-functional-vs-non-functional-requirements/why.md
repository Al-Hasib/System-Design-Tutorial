# কেন এই Topic গুরুত্বপূর্ণ: Functional vs Non-Functional Requirements

> **এক বাক্যে:** Functional requirements আপনাকে বলে *কী তৈরি করতে হবে*; non-functional requirements আপনাকে বলে *এটা কীভাবে আচরণ করতে হবে* — এবং প্রায় প্রতিটি architectural সিদ্ধান্ত দ্বিতীয় ধরনের requirements দিয়েই চালিত হয়, যা ঠিক সেই ধরনের requirements যা টিমগুলো লিখতে ভুলে যায়।

## এই Idea আসার আগের World

একটি product spec বলে: "Users একটি profile photo আপলোড করতে পারবে।" টিম এটা তৈরি করে। এটা কাজ করে। তারপর:

- কেউ একজন একটি 400 MB TIFF আপলোড করে এবং server-এর memory শেষ হয়ে যায়।
- Mobile-এ আপলোড হতে 45 সেকেন্ড সময় লাগে এবং users flow ছেড়ে চলে যায়।
- Legal জিজ্ঞাসা করে ছবিগুলো কোথায় সংরক্ষিত (stored) আছে, কারণ EU users-এর data EU-এর বাইরে যেতে পারে না।
- Photo service বন্ধ হয়ে যায় এবং সাথে পুরো login page-ও নিয়ে যায়।

এগুলোর প্রতিটিই এমন একটি requirement যা প্রথম দিন থেকেই বাস্তব ছিল কিন্তু কখনো লেখা হয়নি। Feature-টি "সম্পন্ন (done)" ছিল, কিন্তু system তখনও ভুল ছিল।

## এটি যেসব সমস্যা সমাধান করে

### ১. সঠিক feature ভুল properties দিয়ে তৈরি করা
**আপনি যা দেখেন:** Functionality ঠিক যেমন specify করা হয়েছিল ঠিক তেমনই আছে, তবুও এটা ব্যবহারযোগ্য নয় — খুব ধীর, খুব দুর্বল (fragile), অথবা চালাতে খুব ব্যয়বহুল।

**কেন এটা ঘটে:** Spec-গুলো user-visible behavior-এর ভাষায় লেখা হয়, যা functional অংশ। Latency, throughput, availability, durability, এবং cost একটি spec-এ অদৃশ্য থাকে কিন্তু architecture-এ সিদ্ধান্তমূলক (decisive)। একটি "send message" feature-এর design সম্পূর্ণ ভিন্ন হবে, যদি এটাকে 100 ms-এর কম সময়ে deliver করতে হয় বনাম "eventually।"

**এই পার্থক্য কীভাবে এটা সমাধান করে:** এটি আপনাকে প্রতিটি feature-এর জন্য দ্বিতীয় সেট প্রশ্ন জিজ্ঞাসা করতে বাধ্য করে: কত দ্রুত, কতজন, কতটা available, কতটা durable, কতটা secure। এই উত্তরগুলোই — feature list নয় — নির্ধারণ করে আপনার একটি queue, cache, replica, নাকি CDN দরকার।

### ২. এমন Architecture বিতর্ক যার কোনো নিষ্পত্তিকারী নেই
**আপনি যা দেখেন:** "আমাদের Postgres নাকি DynamoDB ব্যবহার করা উচিত?" একটি ধর্মীয়-স্তরের বিতর্কে পরিণত হয়। দুই পক্ষেরই ভালো যুক্তি আছে এবং কেউই এটা বন্ধ করতে পারে না।

**কেন এটা ঘটে:** নির্ধারিত non-functional targets ছাড়া, প্রতিটি option-ই যুক্তিসঙ্গত। Technology-এর পছন্দ শুধুমাত্র *constraints-এর সাপেক্ষে* সিদ্ধান্তযোগ্য।

**এই পার্থক্য কীভাবে এটা সমাধান করে:** "99.99% availability, প্রতি সেকেন্ডে 50k writes, balance updates-এ strong consistency" লিখে ফেলুন এবং option-এর জায়গা দ্রুত সংকুচিত হয়ে যায়। Requirements-ই architecture-কে রুচি (taste) থেকে engineering-এ রূপান্তরিত করে।

### ৩. Scope যা নীরবে তিনগুণ হয়ে যায়
**আপনি যা দেখেন:** একটি দুই-সপ্তাহের feature তিন মাসে ship হয়, কারণ "আমাদের এগুলোও দরকার ছিল" — audit logging, rate limiting, retries, এবং একটি admin tool।

**কেন এটা ঘটে:** Cross-cutting non-functional needs (security, observability, compliance, operability) একটির পর একটি আবিষ্কৃত হয়, build চলাকালীন, যখন প্রতিটিকে retrofit করা সবচেয়ে বেশি ব্যয়বহুল।

**এই পার্থক্য কীভাবে এটা সমাধান করে:** শুরুতেই এগুলোকে first-class requirements হিসেবে সামনে আনার মানে হলো এগুলো হয় বাজেট করা হয়, ইচ্ছাকৃতভাবে পিছিয়ে দেওয়া হয়, অথবা ইচ্ছাকৃতভাবে বাদ দেওয়া হয় — delivery date-কে হঠাৎ আক্রমণ করার বদলে।

### ৪. এমন Interview যেখানে আপনি ভুল system design করেন
**আপনি যা দেখেন:** একজন candidate-কে একটি URL shortener design করতে বলা হয় এবং সে হ্যাশিং অ্যালগরিদমে (hashing algorithm) বিশ মিনিট ব্যয় করে, কখনো জিজ্ঞাসা করে না কতগুলো URL, read/write ratio কেমন, অথবা links-এর মেয়াদ শেষ হয় কিনা।

**কেন এটা ঘটে:** প্রম্পটটি ইচ্ছাকৃতভাবে অস্পষ্ট। Interviewer-রা constraints বাদ দেয়, দেখার জন্য যে আপনি সেগুলো জিজ্ঞাসা করবেন কিনা।

**এই পার্থক্য কীভাবে এটা সমাধান করে:** Requirement clarification দিয়ে শুরু করা — প্রথমে functional, তারপর scale/latency/availability/consistency — একটি design interview-তে একক সর্বোচ্চ-স্কোরিং পদক্ষেপ, কারণ এটা প্রমাণ করে আপনি জানেন যে constraints-ই *হলো* সমস্যা।

## যে মূল্য আপনাকে দিতে হয়

- **Requirement theater।** "system-টি অবশ্যই highly available হবে"-এর মতো কোনো সংখ্যা ছাড়া লম্বা document শূন্যের চেয়েও খারাপ; এগুলো ভুয়া আত্মবিশ্বাস তৈরি করে।
- **আগেভাগে over-specification।** একটাও user থাকার আগেই 99.999% availability-এর প্রতিশ্রুতি দেওয়া, এমন load-এর জন্য ব্যয়বহুল architecture তৈরি করে যা হয়তো কখনোই আসবে না।
- **সময়ের খরচ।** প্রকৃতপক্ষে non-functional requirements বের করার মানে হলো product, legal, এবং ops-এর সাথে কথা বলা — শুধু code লেখার চেয়ে ধীর, এবং প্রায়ই এটা করার যোগ্য।

## কখন এটা দরকার — এবং কখন নয়

| Requirements নির্ধারণ করুন যখন | দ্রুত এগিয়ে যান যখন |
|---|---|
| System টাকা, identity, বা regulated data সামলায় | এটা একটি throwaway prototype বা spike |
| আপনি এমন infrastructure বেছে নিচ্ছেন যা পরিবর্তন করা ব্যয়বহুল | আপনি একটি বিদ্যমান page-এ একটি button যোগ করছেন |
| একাধিক টিম এর সাথে integrate করবে | Blast radius একটি একক internal tool |
| আপনি একটি system design interview-তে আছেন — *সবসময়* | — |

## Interview-এ এটি কেন গুরুত্বপূর্ণ

এটি প্রতিটি system design round-এর প্রারম্ভিক পদক্ষেপ, এবং এটাকে ভারী স্কোর দেওয়া হয়। যে candidate বলে "Design করার আগে, আমাকে স্পষ্ট করতে দিন: কতজন daily active users, read-to-write ratio কেমন, এখানে কি আমাদের strong consistency দরকার, এবং latency target কী?" — সে ইতিমধ্যেই এমন কাউকে এগিয়ে যায় যে সরাসরি boxes আঁকা শুরু করে। পরে আঁকা প্রতিটি জিনিসেরই সেই উত্তরগুলোর একটির সাথে যোগসূত্র থাকা উচিত।

## এটি কীভাবে সংযুক্ত

Non-functional requirements-ই এই কোর্সের বাকি অংশের অস্তিত্বের কারণ। *Scalability* উত্তর দেয় "কতজন", *availability এবং fault tolerance* উত্তর দেয় "কতটা reliable", *caching এবং CDNs* উত্তর দেয় "কত দ্রুত", *consistency models* উত্তর দেয় "কতটা সঠিক (correct)"। পরবর্তী প্রতিটি topic হলো এক শ্রেণীর non-functional requirement সন্তুষ্ট করার একটি tool — তাই requirement-এর নাম জানা মানে জানা কোন tool ব্যবহার করতে হবে।

**পরবর্তী:** [Client-Server Architecture & How the Internet Works](../03-client-server-architecture-and-how-the-internet-works/why.md) — যে ভিত্তির (substrate) উপর প্রতিটি requirement শেষ পর্যন্ত চলে।
</content>
