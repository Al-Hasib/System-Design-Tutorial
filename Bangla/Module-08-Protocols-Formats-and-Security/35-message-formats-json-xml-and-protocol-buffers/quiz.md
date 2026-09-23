# অনুশীলন ও ইন্টারভিউ প্রশ্ন

**১. Serialization কী, এবং কেন প্রতিটি distributed system-এর এর জন্য একটি শেয়ার্ড, সম্মত format প্রয়োজন?**
Serialization in-memory ডেটাকে transmission বা storage-এর জন্য একটি byte sequence-এ রূপান্তর করে। দুটি service একে অপরের byte তখনই বুঝতে পারে যখন উভয়ে আগে থেকেই সম্মত হয় যে সেই byte-গুলো ঠিক কীভাবে structured — এই সম্মতিটাই message format। এটি ছাড়া, প্রাপকের কাছে প্রেরকের ডেটা পুনর্গঠন করার কোনো নির্ভরযোগ্য উপায় থাকবে না।

**২. একই logical data-এর জন্য JSON কেন Protocol Buffers-এর চেয়ে বেশি verbose?**
JSON প্রতিটি field name-কে প্রতিটি message-এ একটি text string হিসেবে পুনরাবৃত্তি করে (`"userId"`, `"userName"`, ইত্যাদি), যেখানে Protocol Buffers field name-এর পরিবর্তে একটি shared schema-তে একবার সংজ্ঞায়িত ছোট numeric tag ব্যবহার করে — field name wire-এ কখনোই প্রদর্শিত হয় না, শুধুমাত্র `.proto` definition-এ থাকে যা উভয় পক্ষের কাছেই ইতিমধ্যে আছে।

**৩. Protocol Buffers-এর "schema-first" হওয়ার অর্থ কী, এবং JSON-এর তুলনায় এটি কী সুবিধা দেয়?**
Message structure (field name, type, numeric tag) আগে থেকেই একটি `.proto` file-এ সংজ্ঞায়িত থাকে, এবং প্রেরক ও প্রাপক উভয়েই সেই একই schema থেকে strongly-typed code generate করে। এটি compile time-এ type mismatch ধরে ফেলে এবং নিশ্চিত করে যে উভয় পক্ষই structure নিয়ে সম্মত, যেখানে JSON-এর field loosely typed এবং enforced নয় যদি না আপনি JSON Schema-র মতো একটি আলাদা validation layer যোগ করেন।

**৪. কেন XML আজকাল নতুন API design-এর বদলে enterprise/legacy system-এ বেশি দেখা যায়?**
XML শক্তিশালী native schema এবং validation support (XSD) এবং namespace প্রদান করে, যা কঠোর contract সহ বড় enterprise integration-এর জন্য গুরুত্বপূর্ণ ছিল, কিন্তু এর tag-based syntax সাধারণ ক্ষেত্রে JSON-এর চেয়ে বেশি verbose। বেশিরভাগ নতুন system এখন JSON-এর সরলতা এবং ছোট payload পছন্দ করে, যার ফলে XML মূলত legacy system এবং নির্দিষ্ট enterprise/SOAP integration-এ থেকে যায় যার সাথে আপনাকে হয়তো এখনও interoperate করতে হয়।

**৫. Protocol Buffers-এ, একটি message-এ নতুন field যোগ করাকে "নিরাপদ" এবং একটি বিদ্যমান field-এর number পরিবর্তন করাকে "অনিরাপদ" করে তোলে কী?**
একটি নতুন, অব্যবহৃত tag number সহ নতুন field যোগ করা নিরাপদ কারণ পুরনো code সেই tag যা চেনে না তা কেবল উপেক্ষা করে। একটি বিদ্যমান field-এর tag পুনরায় ব্যবহার বা পুনঃসংখ্যায়ন করা অনিরাপদ কারণ এটি পুরনো এবং নতুন code-কে সেই tag-এর অর্থ নিয়ে দ্বিমত করে তোলে — একটি পক্ষ সেই field-এর জন্য সম্পূর্ণ ভুল ডেটা পড়তে পারে।

**৬. আপনি কেন JSON বা XML-এর ভেতরে সরাসরি raw binary data (যেমন একটি image) embed করতে পারবেন না?**
উভয়ই text format, তাই binary byte-কে প্রথমে একটি text-safe representation-এ encode করতে হয় — সাধারণত base64 — যা ডেটার size-এ প্রায় ৩৩% যোগ করে। Protocol Buffers, native ভাবে একটি binary format হওয়ায়, এই encoding overhead ছাড়াই সরাসরি raw byte অন্তর্ভুক্ত করতে পারে।

**৭. পরিস্থিতি: আপনি একটি public API design করছেন যার সাথে third-party developer-রা integrate করবে, যাদের অনেকেই curl বা Postman দিয়ে হাতে debug করবে। আপনি কোন message format বেছে নেবেন, এবং কেন?**
JSON — এটি human-readable, ভাষা এবং tool জুড়ে সার্বজনীনভাবে সমর্থিত, এবং developer-দের আপনার schema definition বা generated code ছাড়াই সরাসরি response পরিদর্শন এবং debug করতে দেয়, যা একটি public integration surface-এর জন্য অনেক গুরুত্বপূর্ণ।

**৮. পরিস্থিতি: আপনার মালিকানাধীন দুটি service প্রতিদিন একে অপরকে কয়েক মিলিয়ন call করে, এবং আপনি লক্ষ্য করেছেন profiling-এ serialization overhead দেখা যাচ্ছে। আপনি কী পরিবর্তন করার কথা ভাববেন, এবং কেন?**
JSON/REST থেকে gRPC-এর উপর Protocol Buffers-এ স্থানান্তরিত হওয়া — যেহেতু উভয় service internal এবং আপনার নিয়ন্ত্রণে, আপনি `.proto` schema এবং generated code শেয়ার করতে পারেন, এবং ছোট binary payload প্লাস দ্রুততর (de)serialization সেই call volume-এ bandwidth এবং CPU সময় উভয়ই কমাবে।

**৯. Schema evolution কী, এবং স্বাধীনভাবে deploy করা service-গুলোর জন্য এটি কেন গুরুত্বপূর্ণ?**
Schema evolution হলো সময়ের সাথে একটি message format পরিবর্তন করার (field যোগ, অপসারণ, বা পরিবর্তন) নিয়ম/পদ্ধতির একটি সেট, যাতে একই সাথে পুরনো ও নতুন version চালানো service-গুলোর মধ্যে compatibility না ভাঙে। এটি গুরুত্বপূর্ণ কারণ স্বাধীনভাবে-deploy করা service-গুলোকে সবসময় একসাথে upgrade করা যায় না — একটি producer এবং consumer rollout-এর সময় কিছু সময়ের জন্য ভিন্ন schema version চালাতে পারে, এবং format-কে তা সহ্য করতে হবে।

**১০. সত্য নাকি মিথ্যা: JSON নিশ্চিত করে যে একটি নির্দিষ্ট field সব message জুড়ে সবসময় একই data type ধারণ করে।**
মিথ্যা। JSON নিজে কোনো schema enforcement করে না — একটি field প্রযুক্তিগতভাবে একটি message-এ string এবং আরেকটিতে number ধারণ করতে পারে যদি না application layer (অথবা JSON Schema-র মতো একটি অতিরিক্ত tool) তা validate করে। বিপরীতে, Protocol Buffers schema এবং generated-code স্তরে field type enforce করে।
</content>
