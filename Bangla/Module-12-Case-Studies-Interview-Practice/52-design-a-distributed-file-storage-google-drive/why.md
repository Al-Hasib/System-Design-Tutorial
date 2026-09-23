# কেন এই বিষয়টি গুরুত্বপূর্ণ: একটি Distributed File Storage System ডিজাইন করা (Google Drive/Dropbox-এর মতো)

> **এক বাক্যে:** এটি সেই case study যেখানে metadata এবং content আলাদা করতে হয়, যেখানে এক-byte-এর edit-এর জন্য এক gigabyte আবার upload করা চলবে না, এবং যেখানে দুইজন মানুষ offline অবস্থায় একই file edit করলে এমন একটি conflict প্রশ্নের উত্তর দিতে বাধ্য করে যার কোনো পরিষ্কার technical সমাধান নেই।

## এই Case Study-টা কেন আছে

বেশিরভাগ design সমস্যা ছোট record নিয়ে কাজ করে। এটি বড়, opaque binary object নিয়ে কাজ করে — এবং এটি storage, transfer, এবং consistency সম্পর্কে সবকিছু পরিবর্তন করে দেয়।

এটি course-এর সবচেয়ে স্পষ্ট উদাহরণ যেখানে **metadata সমস্যা এবং data সমস্যা সম্পূর্ণ ভিন্ন সমস্যা**। File metadata (নাম, folder structure, permission, version, sharing) ছোট, highly relational, ঘন ঘন query হওয়া, এবং transaction প্রয়োজন। File content বিশাল, একবার লেখা হলে immutable, এবং সস্তা durable storage এবং দ্রুত delivery প্রয়োজন। দুটোকেই একটি single system দিয়ে serve করার চেষ্টা করলে এমন কিছু তৈরি হয় যা দুটোতেই খারাপ — এই বিভাজন প্রথমেই চিনে নেওয়াই এই সমস্যার প্রথম প্রকৃত insight।

## যেসব Design সমস্যা এটি আপনাকে সমাধান করতে বাধ্য করে

### ১. বড় file efficiently upload এবং sync করা
**সমস্যা:** একজন user একটি 2 GB video project-এর একটি paragraph edit করে। পুরো 2 GB আবার upload করা অগ্রহণযোগ্য। একটি অস্থির connection-এর উপর 5 GB upload শূন্য থেকে restart করা চলবে না।

**কেন এটা কঠিন:** File-গুলো opaque blob; naive system তাদের atomic হিসেবে treat করে।

**যা আপনি শিখবেন:** **Chunking।** File-গুলোকে fixed বা content-defined block-এ ভাগ করুন (4 MB একটি সাধারণ পছন্দ), প্রতিটি hash করুন, এবং file-কে chunk hash-এর একটি ordered list হিসেবে treat করুন। এখন একটি ছোট edit মানে একটি chunk upload। একটি interrupted upload শেষ acknowledged chunk থেকে resume হয়। Sync hash list তুলনা করে এবং শুধু পার্থক্যটুকু transfer করে। এই একটিমাত্র technique-ই এই product-কে সম্ভব করে তোলে, এবং content-defined chunking (boundary content দ্বারা নির্ধারিত, offset দ্বারা নয়) হলো সেই refinement যা edit-কে পরবর্তী প্রতিটি boundary shift করা থেকে আটকায়।

### ২. একই content একবার store করা
**সমস্যা:** দশ হাজার employee-র একই onboarding PDF আছে। একজন user এমন একটি file upload করে যা তার অন্য folder-এ ইতিমধ্যে আছে।

**যা আপনি শিখবেন:** **Content-addressed deduplication** — প্রতিটি chunk তার hash-এর অধীনে store করুন, এবং identical content-এর দ্বিতীয় upload শুধুই একটি reference। এটি একটি বড় বাস্তব cost saving, এবং এর সাথে উল্লেখযোগ্য কিছু consequence আসে: deletion-এর জন্য এখন reference counting দরকার, এবং cross-user deduplication-এর একটি privacy side channel আছে (upload speed observe করে আপনি জানতে পারবেন একটি file ইতিমধ্যে আছে কিনা), যে কারণে কিছু system শুধু একজন user-এর নিজস্ব account-এর মধ্যেই deduplicate করে।

### ৩. Metadata-কে content থেকে আলাদা করা
**সমস্যা:** একটি folder list করা instant হতে হবে; এক petabyte store করা সস্তা হতে হবে।

**যা আপনি শিখবেন:** Metadata একটি database-এ যায় — relational একটি ভালো fit, কারণ folder hierarchy, sharing permission, এবং version history সত্যিকার অর্থেই relational এবং transaction থেকে উপকৃত হয়। Content object storage-এ (S3-style) যায়, যা durability এবং সস্তা bulk capacity-র জন্য তৈরি। Client metadata service-এর সাথে কথা বলে জানতে যে file-টি *কী*, এবং তারপর একটি presigned URL ব্যবহার করে সরাসরি object storage-এ chunk upload বা download করে — **byte-গুলোকে আপনার application server থেকে সম্পূর্ণ দূরে রাখা**, যা একটি workable design-কে সেই design থেকে আলাদা করে যেখানে আপনার API tier একটি bandwidth bottleneck হয়ে যায়।

### ৪. দুটো device offline edit করলে Conflict
**সমস্যা:** একজন user একটি document একটি laptop-এ কোনো connectivity ছাড়াই edit করে, যখন একজন সহকর্মী অন্য কোথাও একই file edit করে। দুজনেই online আসে।

**কেন এটা কঠিন:** নির্বিচারে binary content-এর জন্য technically সঠিক কোনো merge নেই, এবং wall-clock timestamp-এর ভিত্তিতে "last write wins" বেছে নেওয়া clock skew-এর ভিত্তিতে কারো কাজ নীরবে ধ্বংস করে দেয়।

**যা আপনি শিখবেন:** এটি একটি **architecture হিসেবে প্রকাশিত product সিদ্ধান্ত**। Version vector শনাক্ত করে যে দুটো edit sequential না হয়ে concurrent ছিল; এরপর আপনি কী করবেন — উভয়কেই "Document (conflicted copy)" হিসেবে রাখা, user-কে prompt করা, বা পরিচিত format-এর জন্য merge করা — সেটা user experience সম্পর্কে একটি পছন্দ। Detection এবং resolution যে আলাদা জিনিস, এবং detection-এর জন্য timestamp নয় বরং logical clock দরকার — এটা স্পষ্টভাবে বলাটাই এখানে high-signal উত্তর।

### ৫. প্রতিটি device-এ পরিবর্তন propagate করা
**সমস্যা:** একটি device-এ একটি পরিবর্তন user-এর অন্যান্য device-এ এবং collaborator-দের device-এ দ্রুত দেখানো উচিত, প্রতিটি client ক্রমাগত poll না করেই।

**যা আপনি শিখবেন:** একটি notification channel (long-lived connection বা push) যা client-দের বলে "কিছু পরিবর্তন হয়েছে, delta fetch করতে এসো," এর সাথে একটি per-user monotonically increasing change cursor যাতে client জিজ্ঞেস করতে পারে "sequence 4821-এর পর থেকে কী পরিবর্তন হয়েছে?" এটি পুরো tree diff করার চেয়ে অনেক বেশি efficient এবং এভাবেই প্রকৃত sync engine কাজ করে।

## ভুল করলে যা খেসারত দিতে হয়

- **File-কে atomic blob হিসেবে treat করা** মানে কোনো resumable upload নেই, কোনো delta sync নেই, এবং বাস্তব network-এ অব্যবহারযোগ্য একটি product।
- **File byte-কে আপনার API server-এর মধ্য দিয়ে route করা** আপনার application tier-কে একটি bandwidth-bound cost center-এ পরিণত করে; presigned direct-to-object-storage হলো standard fix।
- **File content-কে একটি relational database-এ রাখা** classic ভুল উত্তর এবং প্রতিটি dimension-এ ব্যয়বহুল।
- **Timestamp দিয়ে conflict resolve করা** নীরবে user data হারায় — এমন একটি failure mode যা user-রা কখনো ক্ষমা করে না।
- **Deduplication-এর সাথে reference counting ভুলে যাওয়া** মানে একজন user-এর file delete করা আরেকজন user-এর data ধ্বংস করে দেয়।

## কেন Interviewer-রা এটা বেছে নেন

এটি আলোচনাকে request/response throughput থেকে সরিয়ে storage architecture, data transfer efficiency, এবং offline-first synchronization-এর দিকে নিয়ে যায় — feed এবং chat সমস্যাগুলো থেকে সম্পূর্ণ ভিন্ন একটি skill set। এর একটি অস্বাভাবিকভাবে পরিষ্কার layered structure-ও আছে (client sync engine, metadata service, block storage, notification service), যা একজন candidate একটি system-কে স্পষ্ট responsibility সহ service-এ ভাগ করতে পারে কিনা তার একটি ভালো পরীক্ষা। এবং conflict প্রশ্নটি নির্ভরযোগ্যভাবে সেইসব candidate-কে আলাদা করে যারা timestamp-এর দিকে হাত বাড়ায় তাদের থেকে যারা জানে কেন সেটা নিরাপদ নয়।

## এটি কীভাবে সংযুক্ত

এই case study metadata-বনাম-content বিভাজনের জন্য **object storage এবং SQL/NoSQL নির্বাচন** (topic 11), **replication এবং durability** (topic 13), user দ্বারা metadata-র **sharding** (topic 14), download acceleration-এর জন্য **CDN** (topic 18), conflict detection-এর জন্য **logical clock এবং vector clock** (topic 41), resumable chunk upload-এর জন্য **idempotency** (topic 29), thumbnailing ও virus scanning-এর মতো asynchronous processing-এর জন্য **queue** (topic 20), change notification-এর জন্য **WebSocket বা push** (topic 10), এবং encryption at rest ও sharing permission-এর জন্য **security** (topic 36) প্রয়োগ করে।

**পরবর্তী:** [একটি Video Streaming Platform ডিজাইন করা](../53-design-a-video-streaming-platform-youtube-netflix/why.md) — যেখানে content আরও বড়, এবং delivery network-ই হয়ে ওঠে product।
