# এই টপিকটি কেন গুরুত্বপূর্ণ: ACID vs BASE, Normalization vs Denormalization

> **এক বাক্যে:** এগুলো হলো সেই দুটি নব (knob) যা নির্ধারণ করে আপনার data *গঠনগতভাবে সঠিক* নাকি *গঠনগতভাবে দ্রুত* — এবং খরচ না বুঝে যেকোনো একটিকে ঘোরানোই হলো সেই কারণ যার ফলে সিস্টেমে duplicate charge বা ব্যাখ্যাতীত অসামঞ্জস্য দেখা দেয়।

## এই ধারণার আগের পৃথিবী

দুটি পরিস্থিতি, দুটোই বাস্তব:

**ACID ছাড়া।** একটি ট্রান্সফার অ্যাকাউন্টগুলোর মধ্যে টাকা সরায়: একটিকে debit করা, আরেকটিকে credit করা। প্রক্রিয়াটি দুই ধাপের মাঝখানে ক্র্যাশ করে। টাকা উধাও — একটি অ্যাকাউন্ট থেকে debit হয়েছে, কিন্তু অন্যটিতে কখনো credit হয়নি। কোনো error দেখানো হয়নি। কেউ খেয়ালই করেনি যতক্ষণ না একজন গ্রাহক কল করে।

**Denormalization ছাড়া (বড় স্কেলে)।** একটি social feed query users, posts, follows, likes, এবং media-কে পাঁচটি table জুড়ে জয়েন করে, প্রতি পেজ লোডে চল্লিশ বার, প্রতিদিন এক কোটি ব্যবহারকারীর জন্য। Join গুলো সঠিক, আর database জ্বলছে।

দুটি ব্যর্থতাই সিদ্ধান্ত না নেওয়ার ফল। ACID এবং normalization হলো সেই ডিফল্ট যা correctness-কে সহজ করে তোলে; BASE এবং denormalization হলো সেই পালানোর পথ যা আপনি ইচ্ছাকৃতভাবে নেন যখন ডিফল্ট আর টিকতে পারে না।

## এটি যেসব সমস্যা সমাধান করে

### ১. Partial write যা state নষ্ট করে দেয়
**আপনি যা দেখেন:** একটি অর্ডার আছে কিন্তু তাতে কোনো line item নেই। একটি inventory count কমে গেছে কিন্তু অর্ডারটি ব্যর্থ হয়েছে। টাকা debit হয়েছে কিন্তু credit হয়নি।

**কেন এটা ঘটে:** atomicity ছাড়া multi-step অপারেশন। ধাপগুলোর মাঝে যেকোনো crash, timeout, বা error data-কে এমন একটি state-এ রেখে দেয় যা business logic অসম্ভব বলে মনে করে — আর বাকি কোড, যা এটাকে অসম্ভব ধরে লিখিত, তখন অপ্রত্যাশিতভাবে আচরণ করে।

**ACID কীভাবে এটা সমাধান করে:** Atomicity মানে সব ধাপ commit হয় নয়তো একটিও হয় না। Durability মানে একটি committed transaction power cut-এর পরেও টিকে থাকে। একসাথে এগুলোর মানে হলো আপনাকে কখনো অর্ধ-সম্পন্ন অপারেশনের জন্য recovery code লিখতে হয় না, যা বিনামূল্যে পাওয়া বিশাল পরিমাণ complexity।

### ২. সমান্তরাল অপারেশন যা দুটোই "সফল" হয় এবং দুটোই ভুল
**আপনি যা দেখেন:** একটি কনসার্টের টিকিট, দুটি নিশ্চিতকরণ। দুইজন ব্যবহারকারী উভয়েই "১টি বাকি" পড়েন, উভয়ে কমান, উভয়ে commit করেন।

**কেন এটা ঘটে:** isolation ছাড়া, সমান্তরাল transaction গুলো এমনভাবে মিশে যায় যা কারোরই logic প্রত্যাশা করেনি। এটা কোনো বিরল race নয় — যেকোনো বাস্তব ভলিউমে এটা নিয়মিত ঘটে।

**ACID কীভাবে এটা সমাধান করে:** Isolation সমান্তরাল transaction গুলোকে এমনভাবে আচরণ করায় যেন তারা একে একে চলেছে। আপনি যে level বেছে নেন তা নির্ধারণ করে কতটা কঠোরভাবে, এবং throughput-এ এর খরচ কত।

### ৩. একই তথ্য নয় জায়গায় সংরক্ষিত, তিন জায়গায় ভিন্নমত
**আপনি যা দেখেন:** একজন ব্যবহারকারী তার display name পরিবর্তন করেন। এটা তার প্রোফাইলে আপডেট হয় কিন্তু তার পুরনো comment-এ না, search index-এ না, আর যে notification পাঠানো হয়েছিল তাতেও না।

**কেন এটা ঘটে:** Denormalization read speed-এর জন্য data নকল করে। প্রতিটি কপি এমন একটি জিনিস যা stale হতে পারে। একটি শৃঙ্খলাবদ্ধ update path ছাড়া, এগুলো নীরবে বিচ্যুত হয়ে যায়।

**Normalization কীভাবে এটা সমাধান করে:** প্রতিটি তথ্য ঠিক একবার সংরক্ষণ করুন; বাকি সবকিছু join-এর মাধ্যমে বের করুন। Update গুলো একটি row স্পর্শ করে আর পুরো সিস্টেম সাথে সাথে consistent হয়ে যায়। এই কারণেই normalization সঠিক ডিফল্ট — এই কারণে নয় যে duplication space নষ্ট করে (তেমন করে না, খুব বেশি নয়), বরং এই কারণে যে duplication *inconsistency* তৈরি করে।

### ৪. সঠিক query যা অনেক বেশি ধীর
**আপনি যা দেখেন:** ছয়টি table জুড়ে একটি dashboard join করতে ৪ সেকেন্ড লাগে। এটা নিখুঁতভাবে normalized আর নিখুঁতভাবে অব্যবহারযোগ্য।

**কেন এটা ঘটে:** Join-এর প্রকৃত কাজের খরচ আছে, এবং বড় স্কেলে — বিশেষত shard জুড়ে, যেখানে join অসম্ভবও হতে পারে — সেই খরচই bottleneck হয়ে ওঠে।

**Denormalization কীভাবে এটা সমাধান করে:** join করা shape-টি আগে থেকে গণনা করে সংরক্ষণ করুন যাতে read গুলো single-key lookup হয়। এটাই read-optimized সিস্টেমের ভিত্তি: precomputed feed, materialized view, এবং NoSQL store-এর প্রয়োজনীয় query-first data modeling। খরচ হলো এখন কপিগুলোকে সিঙ্কে রাখার দায়িত্ব আপনার, যা ঠিক BASE-এর trade-off।

## যে মূল্য আপনাকে দিতে হয়

- **ACID-এর খরচ throughput, আর এটা মেশিন জুড়ে ভালোভাবে চলে না।** Locking এবং coordination concurrency সীমিত করে, আর shard বা service জুড়ে distributed transaction ধীর এবং ভঙ্গুর। এই কারণেই BASE-এর অস্তিত্ব।
- **BASE মানে আপনাকে ঝামেলা সামলাতে হবে।** "Eventually consistent" মানে এমন একটি সময়ের জানালা যেখানে সিস্টেম দৃশ্যমানভাবে ভুল। আপনার অ্যাপ্লিকেশনকে তা সহ্য করতে হবে, আর আপনার ব্যবহারকারীদেরও। এর মানে idempotency এবং conflict resolution আপনার সমস্যা হয়ে দাঁড়ায়।
- **Normalization-এর খরচ read performance।** বেশি join, বেশি latency, আর একটি sharded সিস্টেমে, এমন join যা মোটেও চালানো যায় না।
- **Denormalization-এর খরচ write complexity এবং correctness risk।** প্রতিটি write সব কপিতে ছড়িয়ে পড়ে। একটি path মিস করলে — একটি batch job, একটি admin tool, একটি data migration — আপনার কাছে থেকে যায় স্থায়ী নীরব drift।

সৎ সারসংক্ষেপ: **ডিফল্ট হিসেবে normalize করুন এবং ACID ব্যবহার করুন; নির্দিষ্ট, চিহ্নিত bottleneck-এ denormalize করুন এবং BASE-এ শিথিল হন, কপিগুলো সিঙ্কে রাখার একটি লিখিত পরিকল্পনাসহ।**

## কখন এটা দরকার — আর কখন দরকার নেই

| কখন ACID + normalized বেছে নেবেন | কখন BASE + denormalized বেছে নেবেন |
|---|---|
| টাকা, inventory, booking, identity | Feed, timeline, counter, analytics, log |
| Correctness ব্যর্থতা ব্যয়বহুল বা বেআইনি | কয়েক সেকেন্ডের staleness ব্যবহারকারীদের কাছে অদৃশ্য |
| Write ভলিউম আরামে একটি primary-তে ধরে যায় | Write বা read ভলিউমের জন্য horizontal scale দরকার |
| Query pattern ক্রমাগত পরিবর্তিত হতে থাকবে | Query pattern জানা, নির্দিষ্ট, এবং read-প্রধান |

বেশিরভাগ বাস্তব সিস্টেম দুটোই ব্যবহার করে, ভিন্ন ভিন্ন জায়গায়। এটাই উত্তর, কোনো এড়িয়ে যাওয়া নয়।

## এটা কেন Interview-এ আসে

Interviewer-রা এটা যাচাই করেন দেখার জন্য যে আপনি প্রতিটি use case অনুযায়ী guarantee প্রয়োগ করেন নাকি একটি ব্ল্যাংকেট পলিসি হিসেবে। সবচেয়ে শক্তিশালী উত্তর মিশ্র হয়: "Payments Postgres-এ ACID — আমি বরং একটি transaction ব্যর্থ হতে দেব, দুইবার চার্জ করার চেয়ে। Activity feed eventually consistent এবং একটি precomputed timeline-এ denormalized করা, কারণ দুই সেকেন্ড stale একটি feed ঠিক আছে আর read time-এ join করা স্কেল করবে না।" এই বাক্যটি প্রমাণ করে আপনি দুই দিকই বোঝেন *এবং* এই সিদ্ধান্তটি scoped, যা প্রকৃত দক্ষতা।

## এটা কীভাবে সংযুক্ত

এটা হলো data modeling হিসেবে প্রকাশিত **CAP**: BASE হলো AP শাখাটি concrete আকারে। এটা সরাসরি অনুসরণ করে **SQL vs NoSQL** থেকে (NoSQL store সাধারণত denormalized, query-first model প্রয়োজন করে) এবং **sharding** থেকে (যা cross-shard join এবং transaction সরিয়ে দেয়, denormalization-কে বাধ্য করে)। **Distributed transactions — 2PC এবং Saga** হলো সার্ভিস জুড়ে ACID-এর মতো আচরণ পুনরুদ্ধার করার প্রচেষ্টা। **Transaction isolation level** ACID-এর "I"-তে আরও গভীরে যায়, আর **idempotency** হলো সেই টুল যা BASE সিস্টেমগুলোকে retry-এর জন্য নিরাপদ করে তোলে।

**পরবর্তী:** [Caching Strategies & Cache Invalidation](../../Module-04-Caching-and-Content-Delivery/17-caching-strategies-and-cache-invalidation/why.md) — database একেবারেই স্পর্শ না করার সবচেয়ে দ্রুত উপায়।
