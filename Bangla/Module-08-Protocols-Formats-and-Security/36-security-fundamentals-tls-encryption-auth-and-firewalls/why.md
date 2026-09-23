# এই টপিকটি কেন গুরুত্বপূর্ণ: Security Fundamentals — TLS, Encryption, AuthN/AuthZ & Firewalls

> **এক বাক্যে:** Security হলো একমাত্র non-functional requirement যেখানে failure mode হলো degraded service নয় বরং একটি headline, একটি regulatory fine, এবং এমন user যাদের data চিরস্থায়ীভাবে তাদের নিয়ন্ত্রণের বাইরে চলে যায় — এবং প্রায় প্রতিটি breach-ই কয়েকটি সীমিত সংখ্যক fundamentals ভুলভাবে করার সাথে সম্পর্কিত।

## এই আইডিয়ার আগের পৃথিবী

কোনো security design ছাড়া, শুধুমাত্র feature দিয়ে তৈরি একটি সিস্টেম বিবেচনা করুন:

- ট্রাফিক plain HTTP, তাই একই Wi-Fi-তে থাকা যে কেউ session cookie পড়তে পারে এবং user-দের impersonate করতে পারে।
- Password MD5 hash হিসেবে সংরক্ষিত, তাই একটি database dump কয়েক ঘণ্টার মধ্যে crack হয়ে যায়।
- API চেক করে যে আপনি *logged in* কিনা কিন্তু চেক করে না যে আপনি যে record request করেছেন সেটি *আপনার* কিনা, তাই `?id=1042`-কে `?id=1043`-এ পরিবর্তন করলে অন্য কারো invoice ফিরে আসে।
- Database একটি default password দিয়ে publicly reachable, কারণ development-এর সময় এটি সহজ ছিল।
- একটি JWT-র signature যাচাই না করেই trust করা হয়, তাই যে কেউ একটি forge করতে পারে।

এগুলোর কোনোটিই exotic attack নয়। এগুলো বছরের পর বছর ধরে প্রতিটি breach report-এর সাধারণ শীর্ষে থাকে। অস্বস্তিকর সত্য হলো বেশিরভাগ real-world compromise-এ novel cryptography জড়িত নয় — এগুলোতে fundamentals জড়িত যা কেউ owned করেনি।

## এটি যে সমস্যাগুলো সমাধান করে

### ১. Transit-এ পড়া এবং পরিবর্তনযোগ্য Data
**আপনি যা দেখেন:** Shared network-এ session hijacking, একটি compromised router থেকে injected content, একটি proxy দ্বারা harvest করা credential।

**কেন এটি ঘটে:** Unencrypted ট্রাফিক অনেক intermediary-র মধ্য দিয়ে যায় যা আপনি নিয়ন্ত্রণ করেন না — Wi-Fi access point, ISP, transit provider।

**TLS কীভাবে এটি সমাধান করে:** তিনটি property, এবং এগুলো আলাদা তা জানা মূল্যবান। **Encryption** মানে intermediary-রা এটি পড়তে পারবে না। **Integrity** মানে তারা এটি undetected পরিবর্তন করতে পারবে না। **Authentication**, certificate chain-এর মাধ্যমে, মানে client যাচাই করতে পারে যে সে একটি imposter-এর পরিবর্তে আপনার server-এর সাথে কথা বলছে — এই শেষটিই আসলে man-in-the-middle attack থামায়, এবং এই কারণেই certificate validation "সাময়িকভাবে" কখনো disable করা উচিত নয়।

### ২. "আপনি কে" এবং "আপনি কী করতে পারেন" গুলিয়ে ফেলা
**আপনি যা দেখেন:** একজন authenticated user একটি identifier পরিবর্তন করে অন্য user-দের অন্তর্গত record পড়তে, পরিবর্তন করতে, বা মুছে ফেলতে পারে। এই ধরনের ত্রুটি — broken object-level authorization — বাস্তব API-তে সবচেয়ে common serious vulnerability-র মধ্যে ধারাবাহিকভাবে থাকে।

**কেন এটি ঘটে:** Authentication কেন্দ্রীভূত এবং সঠিকভাবে করা সহজ (একটি middleware token চেক করে)। Authorization per-resource, per-endpoint, এবং দুইশটি handler-এর মধ্যে ঠিক একটিতে ভুলে যাওয়া সহজ।

**AuthN থেকে AuthZ আলাদা করা কীভাবে এটি সমাধান করে:** এগুলোকে আলাদা concern হিসেবে নামকরণ করা দ্বিতীয়টিকে প্রতিটি endpoint-এ একটি checklist item করে তোলে। *Authentication* একবার identity প্রতিষ্ঠা করে; *authorization* অবশ্যই প্রতিটি resource access-এর জন্য মূল্যায়ন করতে হবে, server-এ, authenticated identity-র বিরুদ্ধে — কখনো client-supplied ID-র উপর ভিত্তি করে নয় এবং কখনো শুধু UI-তে enforced নয়।

### ৩. Database breach-এ টিকে থাকা Credential
**আপনি যা দেখেন:** একটি leaked password table crack করা হয় এবং credential গুলো অন্য service-এর বিরুদ্ধে replay করা হয়, যেখানে অনেক user সেগুলো reuse করেছিল।

**কেন এটি ঘটে:** Fast hash (MD5, SHA-1, plain SHA-256) গতির জন্য designed, যা password-এর জন্য ঠিক ভুল — একটি GPU প্রতি সেকেন্ডে বিলিয়ন চেষ্টা করে।

**সঠিক hashing কীভাবে এটি সমাধান করে:** bcrypt, scrypt, বা Argon2 ইচ্ছাকৃতভাবে ধীর এবং memory-hard, rainbow table পরাজিত করতে একটি per-user salt সহ। এটি একটি instant mass crack-কে একটি infeasible crack-এ পরিণত করে। এটি একটি one-line library choice যা একটি breach-এর ফলাফল সম্পূর্ণভাবে পরিবর্তন করে।

### ৪. সবকিছু সবজায়গা থেকে পৌঁছানোযোগ্য
**আপনি যা দেখেন:** একটি compromised web server একজন attacker-কে database, cache, internal admin panel, এবং message broker-এ সরাসরি network access দেয়।

**কেন এটি ঘটে:** কোনো segmentation ছাড়া একটি flat network। একবার ভেতরে ঢুকলে, একজন attacker কোনো বাধা ছাড়াই laterally move করে।

**Network security কীভাবে এটি সমাধান করে:** Defense in depth। Firewall এবং security group সীমিত করে কোন component কোন port-এ কার সাথে কথা বলতে পারে। Database private subnet-এ থাকে যার কোনো public route নেই। Admin interface-এর জন্য একটি VPN বা bastion প্রয়োজন। লক্ষ্য unbreachable হওয়া নয় — এটি নিশ্চিত করা যে একটি compromise সম্পূর্ণ compromise না হয়ে ওঠে।

## যে মূল্য আপনাকে দিতে হয়

- **Security friction যোগ করে, এবং friction bypass হয়ে যায়।** যে control legitimate কাজকে কষ্টকর করে তা এড়িয়ে যাওয়া হয় — chat-এ paste করা credential, "demo-র জন্য" disabled MFA, একটি firewall rule যা খোলা হয় এবং কখনো বন্ধ করা হয় না। যে control কেউ follow করে না তা কোনো control না থাকার চেয়ে খারাপ, কারণ এটি একটি false confidence তৈরি করে।
- **Latency এবং cost।** TLS handshake round trip যোগ করে, encryption CPU খরচ করে, এবং authorization check প্রতিটি request-এ database lookup যোগ করে। সবই manageable, কোনোটিই free নয়।
- **Operational burden।** Certificate expire হয় (outage-র একটি সত্যিকারের সাধারণ কারণ), key rotate করতে হয়, secret কোথাও নিরাপদে সংরক্ষণ করতে হয়, এবং dependency ক্রমাগত patch করতে হয়।
- **নিজস্ব cryptography তৈরি করা একটি নিশ্চিত ক্ষতি।** Custom auth scheme, hand-built token format, এবং homemade encryption এমনভাবে ব্যর্থ হয় যা exploit না হওয়া পর্যন্ত দৃশ্যমান হয় না। Vetted library এবং established standard ব্যবহার করুন।
- **Security theater একটি বাস্তব ঝুঁকি।** জটিল password rotation policy, security question, এবং unreviewed compliance checklist প্রচেষ্টা খরচ করে খুব কম কাজ করে — এবং গুরুত্বপূর্ণ fundamentals-কে সরিয়ে দিতে পারে।

## কখন এটি প্রয়োজন — এবং কখন নয়

Fundamentals-এর জন্য কোনো "যখন প্রয়োজন নেই" নেই। সর্বত্র TLS, সঠিক password hashing, প্রতিটি resource access-এ authorization, codebase-এর বাইরে secret, এবং patched dependency real user সহ যেকোনো কিছুর জন্য baseline।

যা context-এর সাথে scale করে তা হলো depth:

| আরও invest করুন যখন | Baseline যথেষ্ট যখন |
|---|---|
| আপনি payments, health, বা personal data নিয়ে কাজ করেন | এটি একটি private network-এ একটি internal tool |
| আপনি GDPR, HIPAA, PCI-DSS, SOC 2-র অধীন | Data-র কোনো confidentiality value নেই |
| আপনি একটি high-value target | — |
| আপনি untrusted client-দের কাছে একটি public API expose করেন | — |

## এটি কেন Interview-তে আসে

Security খুব কমই প্রধান প্রশ্ন এবং প্রায়শই একটি নির্ধারণী follow-up: "আপনি কীভাবে request authenticate করেন?", "আপনি secret কোথায় সংরক্ষণ করেন?", "আপনি কীভাবে নিশ্চিত করেন একজন user শুধু তার নিজের data দেখতে পারে?" যে candidate-রা স্বেচ্ছায় এটি উল্লেখ করে — transit এবং at rest-এ TLS, short expiry এবং refresh সহ token, প্রতি-resource server-side চেক করা authorization, auth endpoint-এ rate limiting, একটি managed store-এ secret — তারা আলাদা হয়ে দাঁড়ায় ঠিক এই কারণেই যে এত অনেকে এটি একেবারেই উল্লেখ করে না। Payments বা personal data জড়িত design-এ, security সম্পূর্ণভাবে বাদ দেওয়া জ্ঞানের ঘাটতির চেয়ে judgment-এর ঘাটতি হিসেবে পড়া হয়।

## এটি কীভাবে সংযুক্ত

TLS **transport layer**-এর (topic 33) উপর চলে এবং সাধারণত **reverse proxy** বা **API gateway**-তে (topic 8, 9) terminate হয়, যা এমন একটি জায়গাও যেখানে **rate limiting** (topic 25) authentication endpoint-কে brute force থেকে রক্ষা করে। Gateway-তে Token validation **microservices communication**-এর (topic 31) কেন্দ্রীয়। **Multi-region** এবং **replication** design-কে (topic 47, 13) হিসাব করতে হয় data আইনগতভাবে কোথায় থাকতে পারে, এবং **observability** (topic 43) হলো যা আপনাকে একটি intrusion detect করতে দেয় — এবং একই সাথে একটি জায়গা যেখানে আপনি অসতর্ক হলে secret এবং personal data log-এ leak হয়ে যায়।

**পরবর্তী:** [Transaction Isolation Levels & Concurrency Control](../../Module-09-Database-and-API-Internals/37-transaction-isolation-levels-and-concurrency-control/why.md) — database-এর ভেতরে ফিরে, যেখানে concurrency-র অধীনে correctness নির্ধারিত হয়।
