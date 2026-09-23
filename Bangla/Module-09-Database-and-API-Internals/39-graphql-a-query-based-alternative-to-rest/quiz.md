# Practice & Interview Questions

**১. Over-fetching এবং under-fetching define করুন, এবং ব্যাখ্যা করুন কেন এগুলো REST-এর fixed-shape endpoints-এর সাথে structural সমস্যা।**
Over-fetching হলো client-এর প্রয়োজনের চেয়ে বেশি fields পাওয়া, কারণ একটি REST endpoint প্রতিটি caller-কে একই fixed shape ফেরত দেয়, তাদের আসলে কী দরকার তা বিবেচনা না করেই। Under-fetching হলো একটি endpoint থেকে যথেষ্ট combined data না পাওয়া, যা একাধিক round trip অথবা একটি বিশেষভাবে তৈরি aggregating endpoint-এর প্রয়োজন তৈরি করে। দুটোই ঘটে কারণ REST endpoints-এর একটি fixed response shape থাকে, client-কে সে যা চায় তা নির্দিষ্ট করতে দেওয়ার বদলে।

**২. GraphQL কীভাবে একই সাথে over-fetching এবং under-fetching উভয়ই সমাধান করে?**
Client একটি single query পাঠায় যা একটি endpoint থেকে ঠিক কোন fields দরকার তা নির্দিষ্ট করে, প্রয়োজন অনুযায়ী গভীরভাবে nested। Server ঠিক query-র মতোই shape করা একটি response ফেরত দেয় — কোনো অব্যবহৃত data নেই (over-fetching নেই) এবং একটি round trip-এই সবকিছু আছে যা দরকার (under-fetching নেই)।

**৩. GraphQL-এ N+1 query problem কী, এবং একটি concrete উদাহরণ দিন।**
এটি তখন ঘটে যখন items-এর একটি list এবং প্রতিটির একটি related field resolve করলে, list-এর জন্য একটি query এবং প্রতিটি item-এর জন্য একটি অতিরিক্ত query trigger হয়। উদাহরণ: 20 জন users fetch করা (1টি query) তারপর naive ভাবে প্রতিটি user-এর orders আলাদাভাবে fetch করা (আরও 20টি query) = মোট 21টি query, 2টির বদলে।

**৪. একটি DataLoader/batching pattern কীভাবে N+1 problem ঠিক করে?**
প্রতিটি resolver চলার সাথে সাথে একটি database query চালানোর বদলে, এটি একটি query execution জুড়ে একই ধরনের data-র জন্য সব requests সংগ্রহ করে এবং সেগুলোকে একটি single batched call হিসেবে issue করে — যেমন, ২০টি আলাদা "fetch orders for user X" query-র বদলে একটি "fetch orders for these 20 user IDs" query।

**৫. HTTP-level caching (CDNs, browser cache, `Cache-Control`) GraphQL-এ REST-এর তুলনায় প্রয়োগ করা কেন অনেক বেশি কঠিন?**
REST-এর `GET` requests স্বাভাবিকভাবেই HTTP caching-এর সাথে মানানসই, কারণ প্রতিটি নির্দিষ্ট resource-এর নিজস্ব URL থাকে যার উপর caches key করতে পারে। GraphQL সাধারণত সব queries-এর জন্য একটি single `POST` endpoint ব্যবহার করে, যা HTTP caching infrastructure URL দিয়ে আলাদা করতে বা cache করতে পারে না — এর বদলে production GraphQL systems-এর নিজস্ব caching layer দরকার (client-side normalized caches, persisted queries)।

**৬. API versioning-এর ক্ষেত্রে GraphQL-এর approach REST-এর সাধারণ approach থেকে কীভাবে আলাদা?**
REST প্রায়ই একটি breaking change দরকার হলে URL-এর মাধ্যমে (`/v1/`, `/v2/`) version করে, যা সম্ভাব্যভাবে একাধিক version একসাথে চালায়। GraphQL-এর convention হলো শুধু schema-তে নতুন fields যোগ করে এবং পুরনোগুলো (সরানোর বদলে) deprecate করে versioning এড়িয়ে যাওয়া, যাতে schema evolve হওয়ার সাথে সাথে existing queries কাজ করতে থাকে — যদিও একটি সত্যিকারের breaking change শেষ পর্যন্ত এখনও একটি কঠিন decision দাবি করে।

**৭. একটি GraphQL API-এর জন্য rate limiting একটি REST API-এর চেয়ে structurally কেন কঠিন?**
একটি নির্দিষ্ট endpoint-এ একটি REST request একটি bounded, predictable কাজ represent করে। একটি single GraphQL query arbitrarily গভীরভাবে nest করতে পারে এবং একটি call-এই বিশাল পরিমাণ related data চাইতে পারে, তাই simple per-request rate limiting যথেষ্ট নয় — production GraphQL APIs-এর সাধারণত query complexity analysis বা depth-limiting দরকার হয়, যাতে একটি expensive query অসামঞ্জস্যপূর্ণভাবে বেশি backend resource খরচ করতে না পারে।

**৮. Scenario: একটি company-র iOS, Android, এবং web team আছে, প্রতিটিরই ঘন ঘন পরিবর্তনশীল screens-এ সামান্য ভিন্ন data combination দরকার, সবকিছু একটি backend team দ্বারা সার্ভ করা হয়। আপনি REST নাকি GraphQL সুপারিশ করবেন, এবং কেন?**
GraphQL — এটি ঠিক সেই scenario যার জন্য এটি design করা হয়েছিল: একাধিক independently-evolving client, ভিন্ন, পরিবর্তনশীল data প্রয়োজনসহ, একটি backend schema share করছে। Frontend teams তাদের queries adjust করে নতুন field combination fetch করতে পারে, backend team-কে প্রতিটি client এবং screen-এর জন্য বিশেষভাবে তৈরি aggregating endpoints বানাতে বা maintain করতে বাধ্য না করেই।

**৯. Scenario: একটি company partners-দের integrate করার জন্য একটি simple, public, third-party-facing API তৈরি করছে, যেখানে সহজ caching এবং straightforward documentation-কে প্রাধান্য দেওয়া হচ্ছে। আপনি REST নাকি GraphQL সুপারিশ করবেন, এবং কেন?**
REST — এই workload-টি well-understood, cacheable, per-resource endpoints-কে favor করে, এবং third-party integrators সাধারণত REST-এর সহজ mental model এবং native HTTP caching থেকে উপকৃত হয়, GraphQL-এর অতিরিক্ত flexibility-র বদলে, যা এখানে অপ্রয়োজনীয় caching এবং rate-limiting complexity নিয়ে আসে।

**১০. সত্য নাকি মিথ্যা: একটি GraphQL schema শুধু server-এর জন্যই useful; clients-দের কোন field available আছে তা introspect করার কোনো উপায় নেই।**
মিথ্যা। GraphQL schemas introspectable — clients এবং tools (যেমন GraphiQL বা Apollo-র tooling) autocomplete, type-checking, এবং auto-generated documentation-কে শক্তি দিতে schema-টিকেই query করতে পারে, যা REST APIs-এর তুলনায় GraphQL-এর একটি practical advantage, যেখানে প্রায়ই আলাদাভাবে হাতে maintain করা documentation-এর উপর নির্ভর করতে হয়।
