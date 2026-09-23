# GraphQL: REST-এর একটি Query-ভিত্তিক বিকল্প

**কঠিনতার মাত্রা:** Intermediate

## শেখার লক্ষ্য (Learning Objectives)

- REST-এর fixed-shape endpoints কীভাবে structurally over-fetching এবং under-fetching সমস্যা তৈরি করে তা ব্যাখ্যা করা।
- GraphQL-এর single endpoint এবং client-specified queries কীভাবে উভয় সমস্যার সমাধান করে তা বর্ণনা করা।
- GraphQL resolvers-এ N+1 query problem কী, এবং batching (DataLoader-style) কীভাবে এটি সমাধান করে তা ব্যাখ্যা করা।
- caching, versioning, এবং tooling-এর দিক থেকে GraphQL ও REST তুলনা করা।
- একটি নির্দিষ্ট system-এর জন্য কখন GraphQL-এর অতিরিক্ত complexity গ্রহণযোগ্য, তা সিদ্ধান্ত নেওয়া।

## স্ক্রিপ্ট (Script)

### Hook / Intro

video 6-এ, আমরা REST তৈরি করেছিলাম একটি পরিষ্কার ধারণার উপর ভিত্তি করে: resources, যা URLs দ্বারা চিহ্নিত, এবং HTTP methods দিয়ে manipulate করা হয়। এই model দারুণভাবে কাজ করে যতক্ষণ না আপনার mobile app-এর home screen-এ একসাথে দরকার হয় একজন user-এর নাম, তাদের শেষ তিনটি order-এর totals, এবং তাদের সাম্প্রতিকতম পাঁচটি notification — সব একটি মাত্র screen load-এ। আপনি কি তিনটি আলাদা REST call করবেন? নাকি একটি বিশেষভাবে তৈরি `/home-screen` endpoint বানাবেন যা ঠিক সেই combination-টিই ফেরত দেয়, আর কিছু নয়, এবং এখন সেটাকে চিরকালের জন্য আরেকটি custom endpoint হিসেবে maintain করবেন? ঠিক এই সমস্যাটিই সমাধান করার জন্য GraphQL তৈরি করা হয়েছিল, এবং এটি ঠিক কোন সমস্যার সমাধান করে তা বোঝা — শুধু "GraphQL trendy" এই কারণে নয় — সেটাই আপনাকে একটি fashion choice-এর বদলে একটি বাস্তব trade-off decision নিতে সাহায্য করে।

### সমস্যাটি: Over-Fetching এবং Under-Fetching

REST endpoints-এর একটি fixed shape থাকে: `GET /users/5` সেইসব fields ফেরত দেয় যা server সেই endpoint-এর জন্য আগে থেকেই ঠিক করে রেখেছে — প্রতিবার, প্রতিটি client-এর কাছে, সেই নির্দিষ্ট client-এর আসলে কী দরকার তা বিবেচনা না করেই। এটি দুটি বিপরীত সমস্যা তৈরি করে। **Over-fetching**: একটি mobile app যার শুধু একজন user-এর নাম এবং avatar দরকার, সেটিও পুরো user object পায় — bio, settings, join date, সবকিছু — যা একটি ধীরগতির mobile connection-এ এমন data-র জন্য bandwidth নষ্ট করে যা কেউ ব্যবহারই করবে না। **Under-fetching**: আমাদের hook-এর ঠিক বিপরীত ঘটনা — একটি screen-এর একাধিক resource (user, orders, notifications) থেকে একসাথে data দরকার, এবং একটি একক REST call দিয়ে সবকিছু পাওয়া যায় না, তাই client হয় একাধিক round trip করে (প্রতিটির নিজস্ব latency খরচ সহ), অথবা backend team ঠিক এই screen-টির জন্য একটি one-off aggregating endpoint বানায়, যা পরবর্তী screen-এর জন্য কাজে আসে না যার সামান্য ভিন্ন combination দরকার।

video 9-এর Backend-for-Frontend pattern-টি মনে করুন — REST-world-এ একটি common সমাধান হলো প্রতিটি client type-এর জন্য একটি BFF রাখা, যা পেছন থেকে calls aggregate করে। এটি কাজ করে, কিন্তু এর মানে হলো প্রতিবার একটি client-এর data-র প্রয়োজন সামান্য পরিবর্তন হলেও একটি নতুন backend service (বা endpoint) লেখা এবং maintain করা।

### GraphQL-এর Approach: একটি Endpoint, Client-Specified Shape

GraphQL এই model-টিকে উল্টে দেয়। প্রতিটি নির্দিষ্ট shape ফেরত দেওয়া অনেক URL-এর বদলে, সাধারণত একটিমাত্র **single endpoint** থাকে, এবং client একটি **query** পাঠায় যা ঠিক কোন fields দরকার তা বর্ণনা করে, প্রয়োজন অনুযায়ী যতটা গভীরভাবে nested হওয়া দরকার, একটি মাত্র request-এ:

```
query {
  user(id: 5) {
    name
    avatarUrl
    orders(limit: 3) { total, createdAt }
    notifications(limit: 5) { message, read }
  }
}
```

Server ঠিক query-র মতোই shape করা একটি JSON response ফেরত দেয় — এর বেশিও না, কমও না। Over-fetching অদৃশ্য হয়ে যায় কারণ client শুধু `name` এবং `avatarUrl` চেয়েছিল, পুরো user object নয়। Under-fetching অদৃশ্য হয়ে যায় কারণ client একসাথে user, orders, এবং notifications একটি round trip-এ, একটি request-এ, কোনো বিশেষভাবে তৈরি aggregating endpoint ছাড়াই নিয়ে আসতে পারে। এটি একটি strongly-typed **schema** দ্বারা enforced হয় — প্রতিটি GraphQL API একটি schema publish করে যা ঠিক কোন types, fields, এবং relationships আছে তা define করে, এবং এটিই GraphQL-এর চমৎকার tooling-এর পেছনের শক্তি: autocomplete, in-editor validation, এবং auto-generated documentation — সবকিছুই সরাসরি schema থেকে derive করা, আলাদাভাবে হাতে maintain করা নয় (REST API docs-এর একটি real, দীর্ঘস্থায়ী সমস্যা হলো এটি actual API থেকে out of sync হয়ে যায়)।

### প্রকৃত খরচ: N+1 Problem

GraphQL-এর flexibility-র server side-এ একটি সত্যিকারের, সুপরিচিত খরচ আছে। একটি query-র প্রতিটি field একটি **resolver** function দিয়ে resolve করা হয়, এবং naive ভাবে, একজন user-এর `orders` resolve করা মানে একটি database query, এবং যদি প্রতিটি order-এর line items-ও দরকার হয়, তাহলে সম্ভবত *প্রতিটি order-এর জন্য* একটি অতিরিক্ত query লাগবে — classic **N+1 query problem**: user-এর জন্য 1টি query, তাদের N টি orders-এর জন্য N টি query, এবং তার নিচে সম্ভবত আরও nested query। একটি specific screen-এর জন্য হাতে লেখা একটি REST endpoint-এ এই সমস্যা হতো না, কারণ একজন মানুষ ঠিক সেই endpoint-এর জন্য একটি efficient, purpose-built database query লিখেছিল। GraphQL-এর generality — প্রতিটি field-কে কিছুটা independently resolve করা — এটিকে একটি structural risk বানিয়ে দেয় যা নিয়ে সক্রিয়ভাবে engineer করতে হয়, সাধারণত একটি **batching/dataloader pattern** ব্যবহার করে: প্রতিটি resolver চলার সাথে সাথে একটি database query চালানোর বদলে, পুরো query জুড়ে একই ধরনের data-র জন্য সব requests সংগ্রহ করে একটি single batched database call হিসেবে পাঠানো হয় (যেমন, ২০টি আলাদা "fetch orders for user X" call-এর বদলে "fetch orders for these 20 users")।

### অন্যান্য বাস্তব Trade-offs

GraphQL-এ caching সত্যিকারভাবেই কঠিন। REST-এর `GET` requests স্বাভাবিকভাবেই HTTP caching-এর সাথে মানানসই (CDNs, browser caches, `Cache-Control` headers — সবকিছুই URL দিয়ে keyed) — Module 4-এর caching strategies মনে করুন। GraphQL সাধারণত সবকিছুর জন্য একটি single `POST` endpoint ব্যবহার করে, যা URL-based HTTP caching-কে সম্পূর্ণভাবে অকার্যকর করে দেয়; production GraphQL systems-এর নিজস্ব একটি caching layer দরকার হয় (প্রায়ই client-এ, যেমন Apollo Client-এর normalized cache, অথবা persisted queries-এর মাধ্যমে) HTTP caching বিনামূল্যে পাওয়ার বদলে। Versioning-ও ভিন্নভাবে কাজ করে: REST APIs প্রায়ই breaking changes দরকার হলে URL-এর মাধ্যমে version করে (`/v1/`, `/v2/`); GraphQL-এর convention হলো শুধু নতুন fields যোগ করে versioning সম্পূর্ণভাবে এড়িয়ে যাওয়া (কখনো existing fields সরানো বা পরিবর্তন করা নয়), বরং schema-তে পুরনো fields-কে deprecate করা — যা ভালো কাজ করে যতক্ষণ না একটি সত্যিকারের breaking change এড়ানো অসম্ভব হয়ে পড়ে। আর rate limiting structurally কঠিন: REST-এ, `/users/5`-এ একটি request একটি bounded, predictable কাজ; GraphQL-এ, একটি single query arbitrarily গভীরভাবে nest করতে পারে এবং একটি call-এই বিশাল পরিমাণ data চাইতে পারে, তাই production GraphQL APIs-এর সাধারণত (Module 6-এর) সাধারণ request-based rate limiting-এর উপরে query complexity analysis অথবা depth-limiting দরকার হয়, যাতে একটি expensive query একটি mini denial-of-service-এর মতো আচরণ করতে না পারে।

### বাস্তব-জগতের উদাহরণ

একটি social media app-এর profile screen-এর কথা চিন্তা করুন: নাম, avatar, bio, follower count, এবং user-এর সর্বশেষ 10টি posts তাদের like counts সহ। REST-এ, এটি হয় client থেকে তিন-চারটি আলাদা call, অথবা একটি custom `/profile-screen/:id` endpoint যা এখন কোনো backend team-এর মালিকানায় থাকে এবং একজন designer screen-এ কিছু পরিবর্তন করলেই প্রতিবার সেটাকে update রাখতে হয়। GraphQL-এ, client-এর নিজের query শুধু ঠিক সেই combination-টিই চায় — পরের sprint-এ frontend team "mutual friends"-ও দেখাতে চাইলে কোনো backend change দরকার হয় না; তারা শুধু তাদের query-তে একটি field যোগ করে, যতক্ষণ সেই field ইতিমধ্যে schema-র কোথাও exist করে। এই কারণেই GraphQL প্রথম Facebook (যেখানে এটি উদ্ভূত হয়েছিল)-এর মতো company-তে এবং Netflix-এ জনপ্রিয় হয়েছিল — অনেকগুলো ভিন্ন client team (iOS, Android, web, TV apps), প্রতিটির সামান্য ভিন্ন data প্রয়োজন, একটি shared backend schema-র বিপরীতে independently evolve করছে।

### সংক্ষিপ্তসার (Recap)

REST-এর fixed-shape endpoints over-fetching (অতিরিক্ত অব্যবহৃত data) এবং under-fetching (একাধিক round trip অথবা বিশেষভাবে তৈরি aggregating endpoints) তৈরি করে — এই সমস্যাগুলো GraphQL সমাধান করে client-কে একটি endpoint থেকে ঠিক যে shape-এর data দরকার তা নির্দিষ্ট করতে দিয়ে, একটি strongly-typed schema-র মাধ্যমে সমর্থিত। সেই flexibility বিনামূল্যে নয়: resolvers-এ N+1 query problem-এর ঝুঁকি থাকে (batching/dataloaders দিয়ে সমাধান করা হয়), HTTP-level caching মূলত কাজ করা বন্ধ করে দেয় এবং তার নিজস্ব সমাধান দরকার হয়, এবং rate limiting-এর জন্য সাধারণ per-request counting-এর বদলে query-complexity analysis দরকার হয়। GraphQL তার complexity ঠিক তখনই অর্জন করে যখন আপনার কাছে একাধিক, independently-evolving client থাকে যাদের একটি shared backend-এর বিপরীতে অর্থপূর্ণভাবে ভিন্ন data প্রয়োজন — সব জায়গায় REST-এর default replacement হিসেবে নয়।

### এরপর কী (What's Next)

এর মাধ্যমে database এবং API internals নিয়ে এই module-এর আলোচনা শেষ হলো। পরবর্তী module-এ, আমরা সম্পূর্ণভাবে distributed coordination-এর দিকে focus সরিয়ে নেব — কীভাবে একাধিক node নিরাপদে একটি lock share করে, events কোন order-এ ঘটেছে তা নিয়ে একমত হয়, এবং massive scale-এ সস্তায় "আমি কি আগে এই element-টি দেখেছি?"-এর মতো প্রশ্নের উত্তর দেয়।

## মূল বিষয়গুলো (Key Takeaways)

- REST-এর fixed-shape endpoints over-fetching (অব্যবহৃত data পাঠানো) এবং under-fetching (combined data প্রয়োজনের জন্য একাধিক round trip অথবা বিশেষভাবে তৈরি aggregating endpoints) তৈরি করে।
- GraphQL একটি endpoint এবং একটি strongly-typed schema expose করে; clients ঠিক যে fields এবং nesting চায় তা নির্দিষ্ট করে, এবং response ঠিক সেই shape-এর সাথে মিলে যায়।
- N+1 query problem হলো GraphQL-এর মূল server-side ঝুঁকি — naive resolvers একটি list-এর প্রতিটি item-এর জন্য একটি করে database query trigger করতে পারে — batching/dataloader patterns দিয়ে সমাধান করা হয়।
- GraphQL REST-এর বিনামূল্যের HTTP-level caching হারায় (single POST endpoint) এবং তার নিজস্ব caching layer দরকার হয়, সাথে simple per-request rate limiting-এর বদলে query-complexity/depth limiting দরকার হয়।
- GraphQL তার complexity তখন অর্জন করে যখন একাধিক independently-evolving client-এর একটি shared backend-এর বিপরীতে অর্থপূর্ণভাবে ভিন্ন, পরিবর্তনশীল data প্রয়োজন থাকে — সামগ্রিকভাবে REST-এর replacement হিসেবে নয়।
