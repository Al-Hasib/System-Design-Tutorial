# এই বিষয়টি কেন গুরুত্বপূর্ণ: Message Formats — JSON, XML & Protocol Buffers

> **এক বাক্যে:** Serialization format হলো দুটি সিস্টেমের মধ্যেকার contract, যেগুলো ভিন্ন team দ্বারা ভিন্ন ভাষায় লেখা এবং ভিন্ন সময়ে deploy করা হতে পারে — আর যখন সেই contract-এর কোনো schema থাকে না, তখন প্রতিটি breaking change আবিষ্কৃত হয় production-এ গিয়ে।

## এই ধারণার আগের পৃথিবী

দুটি service JSON আদান-প্রদান করে। কোনো schema নেই, শুধু একটি সম্মত shape আছে যা একটি wiki পাতায় লেখা আছে যেটি চার মাস ধরে আপডেট হয়নি।

একটি producer team `user_id`-এর নাম পরিবর্তন করে `userId` রাখে। তাদের test pass করে — তারা তাদের নিজেদের code-এর উভয় পাশই পরিবর্তন করেছিল। Production-এ, তিনটি consumer service নীরবে `undefined` পেতে শুরু করে। এদের একটি database-এ null লিখতে শুরু করে। দুই দিন ধরে কেউ খেয়াল করে না।

এদিকে, একটি আলাদা সমস্যা: service প্রতি সেকেন্ডে ২,০০,০০০ message সামলায়, প্রতিটি প্রায় ৮০০ byte-এর JSON, দীর্ঘ পুনরাবৃত্ত field name সহ। Profiling দেখায় যে CPU-এর একটি উল্লেখযোগ্য অংশ text parsing-এ ব্যয় হয়, এবং network egress খরচের বেশিরভাগই বারবার প্রেরিত field name-এর কারণে।

## এটি যেসব সমস্যার সমাধান করে

### ১. Production-এ আবিষ্কৃত হওয়া breaking change
**আপনি যা দেখেন:** upstream-এ একটি field-এর নাম পরিবর্তন, type পরিবর্তন, বা অপসারণ, এবং downstream consumer-গুলো নীরবে ব্যর্থ হওয়া বা crash করা।

**কেন এটি ঘটে:** Schema-বিহীন format-এ কোনো contract থাকে না। producer যা পাঠায় তা consumer-রা যা আশা করে তার সাথে মেলে কিনা তা কিছুই validate করে না, এবং কোনো পরিবর্তন incompatible হলে তা সতর্ক করে না।

**Schema-based format এটি কীভাবে সমাধান করে:** Protobuf (এবং Avro, এবং Thrift) স্পষ্ট schema definition আবশ্যক করে। এর থেকে code generate করা হয়, তাই একটি mismatch runtime-এর বিস্ময়ের বদলে একটি compile error হয়ে যায়। একটি schema registry-র সাথে মিলিয়ে, আপনি deploy করার *আগেই* compatibility যাচাই করতে পারেন — যা একটি production incident-কে একটি CI failure-এ রূপান্তরিত করে।

### ২. Field name-এ ব্যয় হওয়া bandwidth এবং CPU
**আপনি যা দেখেন:** পুনরাবৃত্ত key-তে পরিপূর্ণ payload, এবং উচ্চ volume-এ text parsing-এ পরিমাপযোগ্য CPU খরচ।

**কেন এটি ঘটে:** JSON self-describing: প্রতিটি field name প্রতিটি message-এ, text হিসেবে, প্রেরিত হয়, এবং তা text হিসেবেই parse করতে হয়।

**Protobuf কীভাবে এটি সমাধান করে:** Field-গুলো ছোট integer tag দিয়ে চিহ্নিত হয়, value-গুলো binary-এ encode করা হয়, এবং schema যা ইতিমধ্যে বোঝায় তার কিছুই প্রেরণ করা হয় না। Payload সাধারণত ৩-১০ গুণ ছোট হয় এবং parsing যথেষ্ট দ্রুততর। প্রতি সেকেন্ডে ২,০০,০০০ message-এ, এটি প্রকৃত অর্থ এবং প্রকৃত latency।

### ৩. Type ambiguity যা নীরবে ডেটা নষ্ট করে
**আপনি যা দেখেন:** একটি বড় integer ID JavaScript-এ পৌঁছে precision হারায় কারণ JSON number হলো IEEE 754 double। একটি timestamp একটি service-এ string এবং অন্যটিতে epoch integer। একটি field কখনো string আবার কখনো array।

**কেন এটি ঘটে:** JSON-এর চারটি scalar type আছে এবং উদ্দেশ্য প্রকাশ করার কোনো উপায় নেই। "Number" integer, float, এবং currency সবকিছুই কভার করে, খারাপভাবে।

**Typed schema কীভাবে এটি সমাধান করে:** `int64`, `fixed32`, `bytes`, এবং স্পষ্ট enum ambiguity দূর করে। উপরের precision bug একটি সুপরিচিত, বারবার ঘটতে থাকা production সমস্যা যা একটি typed schema সহজেই প্রতিরোধ করে।

### ৪. Schema evolution যার জন্য সমন্বিত deploy প্রয়োজন
**আপনি যা দেখেন:** একটি field যোগ করার মানে হলো producer এবং সব consumer-কে একসাথে deploy করা, কারণ পুরনো consumer অচেনা field প্রত্যাখ্যান করে বা নতুন consumer-এর নতুন field প্রয়োজন হয়।

**কেন এটি ঘটে:** evolution নিয়ম ছাড়া, যেকোনো পরিবর্তন সম্ভাব্যভাবে breaking, তাই team-গুলো প্রতিরক্ষামূলকভাবে সমন্বয় করে।

**Protobuf-এর নিয়ম কীভাবে এটি সমাধান করে:** Field number স্থায়ী; নতুন field default সহ optional হয়; সরানো field number চিরকালের জন্য reserved থাকে। পুরনো code যা চেনে না তা উপেক্ষা করে, এবং নতুন code অনুপস্থিত field সহ্য করে। Producer এবং consumer স্বাধীনভাবে deploy হয় — যা প্রথম স্থানে service থাকার পুরো উদ্দেশ্য।

## আপনাকে যে মূল্য দিতে হবে

- **আপনি এটি পড়তে পারবেন না।** একটি log বা packet capture-এ একটি Protobuf payload হলো অস্বচ্ছ byte। Debugging-এর জন্য schema এবং tooling প্রয়োজন। JSON-এর readability একটি বিশাল, অবমূল্যায়িত operational সুবিধা।
- **Build-time জটিলতা।** Schema file, একটি code generation step, আপনার build-এ generated artifact, এবং schema-গুলোর নিজেদের জন্য একটি distribution mechanism। এই ঘর্ষণই ঠিক কারণ কেন বেশিরভাগ team-এর একটি ছোট internal API-এর জন্য Protobuf ব্যবহার করা উচিত নয়।
- **শৃঙ্খলা বাধ্যতামূলক।** একটি অবসরপ্রাপ্ত field number পুনরায় ব্যবহার করলে ডেটা এমনভাবে নষ্ট হবে যা diagnose করা অত্যন্ত কঠিন। নিয়মগুলো সহজ কিন্তু ক্ষমাহীন।
- **Browser এটি native ভাবে বোঝে না।** Public এবং browser-facing API বাস্তবিকভাবে JSON-এ থেকে যায়।
- **JSON সাধারণত ঠিকই আছে।** সাধারণ volume-এ, JSON parsing আপনার bottleneck নয়, এবং debuggability byte-এর চেয়ে বেশি মূল্যবান। Protobuf-এর দিকে যান যখন আপনি একটি সমস্যা পরিমাপ করেছেন, কারণ এটি benchmark-এ ভালো ফলাফল দেয় বলে নয়।
- **XML এখনও কারণেই টিকে আছে।** XSD validation, namespace, digital signature, এবং সুপ্রতিষ্ঠিত enterprise ও government standard (SOAP, SAML, ISO 20022) মানে হলো XML নির্দিষ্ট regulated integration-এ সঠিক উত্তর — এবং অন্য সব জায়গায় একটি দুর্বল default।

## কখন আপনার এটি প্রয়োজন — এবং কখন নয়

| যখন JSON ব্যবহার করবেন | যখন Protobuf ব্যবহার করবেন | যখন XML ব্যবহার করবেন |
|---|---|---|
| Public এবং browser-facing API | উচ্চ-volume internal service call | আপনাকে একটি বিদ্যমান XML standard-এর সাথে interoperate করতে হবে |
| Byte-এর চেয়ে debuggability বেশি গুরুত্বপূর্ণ | আপনার gRPC প্রয়োজন | আপনার XSD validation বা signed document প্রয়োজন |
| Config file এবং মানুষের হাতে-সম্পাদিত ডেটা | অনেক team জুড়ে কঠোর contract | Enterprise বা নিয়ন্ত্রক integration এটি দাবি করে |
| Volume মাঝারি | Bandwidth বা CPU পরিমাপযোগ্যভাবে একটি bottleneck | — |

## Interview-এ এটি কেন আসে

Format নির্বাচন খুব কমই নিজের একটি প্রশ্ন হয়ে ওঠে, কিন্তু এটি একটি follow-up হিসেবে আসে: "এই service-গুলো কীভাবে communicate করে?" এবং "client-দের না ভেঙে আপনি কীভাবে এই API evolve করবেন?" শক্তিশালী উত্তরটি স্তরভিত্তিক — compatibility এবং debuggability-র জন্য public edge-এ JSON, ভেতরে gRPC-এর উপর Protobuf যেখানে volume তা justify করে — সাথে backward-compatible evolution-এর একটি সুনির্দিষ্ট বিবরণ: শুধুমাত্র additive পরিবর্তন, কখনো একটি field number পুনরায় ব্যবহার নয়, কখনো একটি field-এর অর্থ পুনর্নির্ধারণ নয়। JavaScript-এর বড় integer precision সমস্যাটি উত্থাপন করা একটি চমৎকার, নির্দিষ্ট বিবরণ যা প্রকৃত অভিজ্ঞতার ইঙ্গিত দেয়।

## এটি কীভাবে সংযুক্ত

Protocol Buffers হলো **gRPC** (topic 33)-এর payload format, যা **microservices communication** (topic 31)-এর জন্য একটি সাধারণ পছন্দ। Schema evolution শৃঙ্খলাই **event-driven architecture** (topic 22)-কে টেকসই করে তোলে, যেহেতু event হলো দীর্ঘস্থায়ী contract যা এমন পক্ষরা consume করে যাদের আপনি নাও চিনতে পারেন। JSON হলো **HTTP/REST** (topic 6) এবং **GraphQL** (topic 39)-এর উপর default, এবং payload size সরাসরি **caching** দক্ষতা (topic 17) এবং একটি **BFF**-এর পেছনে mobile performance-কে প্রভাবিত করে (topic 9)।

**পরবর্তী:** [Security Fundamentals](../36-security-fundamentals-tls-encryption-auth-and-firewalls/why.md) — message-গুলোকে, এবং যেসব সিস্টেম সেগুলো আদান-প্রদান করে তাদের রক্ষা করা।
</content>
