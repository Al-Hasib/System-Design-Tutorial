# Message Formats: JSON, XML & Protocol Buffers

**কঠিনতার মাত্রা:** মধ্যম (Intermediate)

## শেখার লক্ষ্যসমূহ (Learning Objectives)

- serialization এবং deserialization কী তা ব্যাখ্যা করা এবং কেন প্রতিটি distributed system-এর একটি সম্মত message format প্রয়োজন তা বোঝা।
- readability, payload size এবং parsing performance-এর দিক থেকে JSON, XML এবং Protocol Buffers-এর তুলনা করা।
- schema কী, কেন Protocol Buffers-এর জন্য schema আবশ্যক, এবং schema evolution (backward/forward compatibility) কীভাবে কাজ করে তা ব্যাখ্যা করা।
- message format-এর নির্বাচন কীভাবে স্কেলে bandwidth, latency, এবং interoperability-কে প্রভাবিত করে তা যুক্তি দিয়ে বোঝা।
- একটি নির্দিষ্ট system design পরিস্থিতির জন্য উপযুক্ত message format বেছে নেওয়া।

## স্ক্রিপ্ট (Script)

### Hook / ভূমিকা

দুটি service-কে ডেটা আদান-প্রদান করতে হবে — ধরা যাক, একটি order service একটি shipping service-কে বলছে "এই যে একটি order, এটা ship করো।" একটি byte পাঠানোর আগেই তাদের একটি বিষয়ে একমত হতে হবে: ডেটাটি আসলে wire-এ কেমন *দেখতে* হবে? এই সম্মতিটাই হলো message format, এবং আপনি কোনটি বেছে নেবেন — JSON, XML নাকি Protocol Buffers-এর মতো কিছু — তার সিস্টেমের bandwidth খরচ, latency, এবং সময়ের সাথে সাথে প্রতিটি client-কে ভেঙে না ফেলে আপনার API কতটা সহজে evolve করা যায় তার উপর বাস্তব, পরিমাপযোগ্য প্রভাব রয়েছে। আজ আমরা এমন তিনটি format তুলনা করব যা আপনি production এবং interview-এ আসলেই সম্মুখীন হবেন, এবং এগুলোর মধ্যে বেছে নেওয়ার জন্য একটি mental model তৈরি করব।

### Serialization: মূল সমস্যা

প্রতিটি প্রোগ্রামিং ভাষা মেমরিতে ডেটা তার নিজস্ব উপায়ে প্রকাশ করে — object, struct, dictionary। **Serialization** হলো সেই in-memory ডেটাকে একটি byte sequence-এ রূপান্তর করার প্রক্রিয়া যা network-এর মাধ্যমে পাঠানো বা disk-এ লেখা যায়। **Deserialization** হলো এর বিপরীত: প্রাপক প্রান্তে সেই byte-গুলোকে আবার ব্যবহারযোগ্য in-memory structure-এ রূপান্তর করা। Message format হলো সেই byte sequence কীভাবে structured হবে তার জন্য একমত হওয়া নিয়মাবলী — যাতে যে কেউ এটি পাঠাক বা গ্রহণ করুক, তারা একেবারে ভিন্ন ভাষায় লেখা হলেও, এটিকে একইভাবে interpret করে।

### JSON: ওয়েবের ডিফল্ট

JSON — JavaScript Object Notation — REST API-এর জন্য de facto মান হয়ে উঠেছে, এবং এর যথাযথ কারণও আছে। এটি human-readable text, এর syntax (object, array, string, number, boolean, null) প্রায় প্রতিটি আধুনিক প্রোগ্রামিং ভাষার data structure-এর সাথে স্বাভাবিকভাবে মিলে যায়, এবং প্রতিটি ভাষারই এটি parse এবং generate করার জন্য পরিপক্ব library আছে। এই readability-ই আবার এর সবচেয়ে বড় খরচও: এটি verbose। প্রতিটি field name প্রতিটি object-এ সম্পূর্ণ string হিসেবে বারবার পুনরাবৃত্তি হয় — `{"userId": 123, "userName": "alice"}` — এবং এতে কোনো built-in schema নেই, তাই কিছুই নিশ্চিত করে না যে `userId` সবসময় একটি number হবে বা কোনো required field আসলেই উপস্থিত থাকবে; সেই validation application layer-এ (অথবা JSON Schema-র মতো একটি অতিরিক্ত schema মান ব্যবহার করে) করতে হয়। JSON-এর integer এবং float-এর মধ্যে পার্থক্য করার কোনো native উপায় নেই, এবং binary data প্রকাশ করার কোনো compact উপায়ও নেই — এটিকে base64-encode করতে হয়, যা প্রায় ৩৩% বেশি byte যোগ করে।

### XML: এন্টারপ্রাইজের প্রবীণ যোদ্ধা

XML — eXtensible Markup Language — ওয়েবের data-interchange format হিসেবে JSON-এর চেয়ে পুরনো এবং এখনও enterprise system, SOAP-based web service, এবং configuration format-এ প্রচলিত। JSON-এর মতোই এটিও human-readable text, তবে এর tag-based syntax (`<user><id>123</id><name>Alice</name></user>`) অনেক বেশি verbose — প্রতিটি value একটি opening এবং closing tag-এ মোড়ানো থাকে। XML-এর প্রকৃত শক্তি হলো schema support: XSD (XML Schema Definition) আপনাকে document structure কঠোরভাবে সংজ্ঞায়িত এবং validate করতে দেয়, এবং namespace আপনাকে বিভিন্ন উৎসের vocabulary-কে নাম-সংঘর্ষ ছাড়াই একত্রিত করতে দেয় — এই বৈশিষ্ট্যগুলো একে বড় enterprise integration-এর জন্য আকর্ষণীয় করে তুলেছিল যেখানে payload size-এর চেয়ে কঠোর contract বেশি গুরুত্বপূর্ণ ছিল। বাস্তবে, আজকাল বেশিরভাগ নতুন সিস্টেম XML-এর চেয়ে JSON বেছে নেয় ঠিক এই কারণেই যে সাধারণ ক্ষেত্রে JSON ছোট এবং সহজতর, এবং XML মূলত legacy system এবং নির্দিষ্ট enterprise/SOAP প্রসঙ্গে দেখা যায় যার সাথে আপনি integrate করছেন, নতুন design পছন্দ হিসেবে নয়।

### Protocol Buffers: Binary এবং Schema-First

Google-এর তৈরি Protocol Buffers ("protobuf") একেবারে ভিন্ন approach নেয়: human-readable text format-এর পরিবর্তে, message-গুলো একটি compact **binary** format-এ serialize করা হয়, এবং JSON-এর মতো schema-optional হওয়ার বদলে এগুলো **schema-first** — আপনি আগে থেকেই একটি `.proto` file-এ আপনার message structure সংজ্ঞায়িত করেন, প্রতিটি field-এর name, type, এবং একটি unique field number উল্লেখ করে। এরপর একটি code generator আপনার ভাষার জন্য strongly-typed class তৈরি করে, ফলে প্রেরক এবং প্রাপক উভয়েই loosely-typed text parse করার বদলে generated, type-safe object নিয়ে কাজ করে।

এতে দুটি বড় সুবিধা পাওয়া যায়। প্রথমত, size এবং speed: যেহেতু field name প্রতিটি message-এ পুনরাবৃত্তি হয় না (binary format এর পরিবর্তে schema থেকে পাওয়া field number ব্যবহার করে), এবং value-গুলো text হিসেবে না রেখে দক্ষভাবে packed করা হয়, তাই protobuf message সাধারণত সমতুল্য JSON-এর চেয়ে কয়েকগুণ ছোট হয়, এবং এগুলো serialize ও parse করা যথেষ্ট দ্রুততর কারণ tokenize করার জন্য কোনো text নেই। দ্বিতীয়ত, contract enforcement: যেহেতু উভয় পক্ষই একই `.proto` schema থেকে code generate করে, "field আসলে একটি string ছিল, number নয়" ধরনের সম্পূর্ণ একটি category-র bug একেবারেই ঘটতে পারে না — এটি production-এ নয়, compile time-এ ধরা পড়ে। এর trade-off ঠিক যেমনটা আশা করা যায়: protobuf message wire-এ human-readable নয় (এগুলো decode করতে schema এবং tooling দরকার), এবং প্রতিটি consumer-এর `.proto` definition এবং generated code-এ access দরকার, যা "শুধু কিছু JSON পাঠিয়ে দাও"-এর তুলনায় process overhead যোগ করে। এই কারণেই gRPC (আগের ভিডিও থেকে) ডিফল্টভাবে Protocol Buffers ব্যবহার করে — এটি দ্রুত, typed, internal service-to-service communication-এর জন্য তৈরি, কোনো অচেনা মানুষ curl করে পড়তে পারবে এমন public API-এর জন্য নয়।

### Schema Evolution

একটি সূক্ষ্ম বিষয় যা বাস্তব সিস্টেমে অনেক গুরুত্বপূর্ণ: দুটি service ইতিমধ্যে production-এ কথা বলছে এমন কয়েক মাস পরে যখন আপনার একটি field যোগ বা পরিবর্তন করতে হয়, সম্ভবত পুরনো এবং নতুন উভয় version একসাথে চলা অবস্থায়, তখন কী ঘটে? JSON এটিকে শিথিলভাবে সামলায় — একটি অচেনা field সাধারণত পুরনো consumer উপেক্ষা করে, এবং অনুপস্থিত field সাধারণত `null` বা absent হয়ে যায়, যা flexible কিন্তু এর অর্থ এই নয় যে কেউ compatibility নিয়ে ভাবতে বাধ্য। Protocol Buffers একে একটি স্পষ্ট, প্রথম-শ্রেণির বিষয় বানায়: প্রতিটি field-এর একটি স্থায়ী numbered tag থাকে, এবং নিয়মগুলো নির্ভুল — আপনি নতুন field যোগ করতে পারেন (পুরনো code যে field number চেনে না তা কেবল উপেক্ষা করে), কিন্তু আপনি কখনো একটি বিদ্যমান field-এর tag পুনরায় ব্যবহার বা পুনঃসংখ্যায়ন করবেন না, এবং একটি field সরানোর অর্থ হলো তার number reserve করে রাখা যাতে এটি কখনো ভুলবশত অন্য field-এ পুনরায় বরাদ্দ না হয়। এটি সঠিকভাবে করাই আপনাকে প্রতিটি consumer জুড়ে সিঙ্ক্রোনাইজড "big bang" rollout ছাড়াই একটি নতুন service version deploy করতে দেয় — যখন আপনার কাছে বিভিন্ন team-এর মালিকানাধীন কয়েক ডজন service থাকে তখন এটি একটি বাস্তব, ব্যবহারিক উদ্বেগ।

### বাস্তব-জগতের উদাহরণ

আবার আমাদের order এবং shipping service-এর কথা ভাবুন। যদি তারা একটি public-facing REST API-এর মাধ্যমে যোগাযোগ করে যার সাথে একটি third-party warehouse partner-কেও integrate করতে হয়, তাহলে JSON হলো বাস্তবসম্মত পছন্দ — যে কেউ payload পড়তে পারে, browser বা curl দিয়ে debug করতে পারে, এবং এটি সর্বত্র কাজ করে। কিন্তু যদি এটি একেবারে internal call হয়, প্রতিদিন লক্ষ লক্ষ বার ঘটে, আপনার নিজের order service এবং আপনার নিজের shipping service-এর মধ্যে — উভয়ই in-house তৈরি, উভয়ই schema পরিবর্তন হলেই একটি shared `.proto` file থেকে code পুনরায় generate করতে সক্ষম — তাহলে gRPC-এর উপর Protocol Buffers সেই আয়তনে serialization CPU সময় এবং network bandwidth উল্লেখযোগ্যভাবে কমায়, এবং schema-first contract mismatch গুলো build time-এই ধরে ফেলে, রাত ২টার incident-এ নয়।

### সংক্ষিপ্তসার (Recap)

JSON verbose কিন্তু সার্বজনীনভাবে readable এবং tooled — public API এবং যেকোনো কিছুর জন্য সঠিক default যা কোনো মানুষকে পরীক্ষা করতে হবে। XML verbose হওয়ার বিনিময়ে শক্তিশালী schema/validation support দেয়, এবং আজকাল মূলত legacy এবং enterprise integration প্রসঙ্গে দেখা যায়। Protocol Buffers human-readability-র বিনিময়ে একটি compact binary format, দ্রুততর (de)serialization, এবং একটি enforced, evolvable schema পায় — যা এগুলোকে high-throughput internal service communication-এর জন্য, বিশেষ করে gRPC-এর সাথে, স্বাভাবিক পছন্দ করে তোলে।

### পরবর্তী কী

আমরা এখন কভার করেছি কীভাবে byte চলাচল করে (transport protocol), কীভাবে একটি server অনেক request সামলায় (concurrency), এবং কীভাবে service-গুলো সেই byte-এর অর্থ নিয়ে একমত হয় (message format)। পরবর্তী ভিডিওতে, আমরা এমন একটি বিষয় নিয়ে এই module শেষ করব যা আমাদের আলোচিত প্রতিটি layer-কে স্পর্শ করে: security — encryption, authentication বনাম authorization, এবং সেই network defense যা এই সবকিছুকে অননুমোদিত ব্যক্তিদের থেকে নিরাপদ রাখে।

## মূল বিষয়সমূহ (Key Takeaways)

- Serialization in-memory ডেটাকে transmission/storage-এর জন্য byte-এ রূপান্তর করে; deserialization এর বিপরীতটি করে — প্রতিটি distributed system-এর উভয় প্রান্তে এটি সামঞ্জস্যপূর্ণভাবে করার জন্য একটি সম্মত format প্রয়োজন।
- JSON verbose কিন্তু human-readable, সার্বজনীনভাবে সমর্থিত, এবং schema-optional — public/browser-facing API-এর ডিফল্ট।
- XML আরও বেশি verbose কিন্তু শক্তিশালী native schema (XSD) এবং namespace support আছে — আজকাল মূলত legacy এবং enterprise/SOAP integration-এ দেখা যায়।
- Protocol Buffers একটি compact binary, schema-first format: ছোট payload, দ্রুততর (de)serialization, এবং compile-time type safety, human-readability হারানো এবং shared `.proto` definition প্রয়োজন হওয়ার বিনিময়ে।
- Schema evolution — নিরাপদে field যোগ করা, field number কখনো পুনরায় ব্যবহার না করা — এটাই হলো যা সময়ের সাথে message format পরিবর্তিত হলেও স্বাধীনভাবে-deploy করা service-গুলোকে compatible রাখে।
</content>
