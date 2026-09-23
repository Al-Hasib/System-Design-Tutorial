# Study Notes: GraphQL vs. REST

## সংজ্ঞা (Definitions)

- **Over-fetching:** একটি response-এ client-এর আসলে যতটা দরকার তার চেয়ে বেশি fields থাকে, যা bandwidth নষ্ট করে।
- **Under-fetching:** একটি single endpoint যথেষ্ট combined data ফেরত দেয় না, যা একাধিক round trip অথবা একটি বিশেষভাবে তৈরি aggregating endpoint-এর প্রয়োজন তৈরি করে।
- **GraphQL:** APIs-এর জন্য একটি query language এবং runtime, যেখানে clients একটি single endpoint থেকে ঠিক যে shape-এর data চায় তা নির্দিষ্ট করে, একটি strongly-typed schema দ্বারা সমর্থিত।
- **Resolver:** একটি function যা একটি GraphQL query-র একটি field-এর জন্য data fetch করার দায়িত্বে থাকে।
- **N+1 query problem:** একটি single batched query-র বদলে, naive resolvers প্রতিটি parent record-এর জন্য একটি query এবং প্রতিটি related child record-এর জন্য একটি করে query (1 + N) trigger করে।
- **DataLoader / batching pattern:** একটি GraphQL query execution-এর মধ্যে একই ধরনের data-র জন্য সব requests সংগ্রহ করে সেগুলোকে একটি single batched backend call হিসেবে issue করা।

## REST বনাম GraphQL

| দিক | REST | GraphQL |
|---|---|---|
| Endpoints | অনেক, প্রতিটি resource/action-এর জন্য একটি | সাধারণত একটি |
| Response shape | প্রতিটি endpoint-এর জন্য fixed | প্রতিটি query-তে client-specified |
| Over/under-fetching | সাধারণ সমস্যা | design অনুযায়ী সমাধান করা |
| Caching | Native HTTP caching (GET + URL + Cache-Control) | কঠিন — single POST endpoint; client-side বা persisted-query caching দরকার |
| Versioning | প্রায়ই URL-এর মাধ্যমে (`/v1`, `/v2`) | Convention: fields যোগ করা, পুরনোগুলো deprecate করা, versioning এড়িয়ে যাওয়া |
| Rate limiting | সহজ, per-request | query complexity/depth analysis দরকার (একটি query arbitrarily nest করতে পারে) |
| Tooling/docs | প্রায়ই হাতে maintain করা, drift হতে পারে | schema থেকে auto-generated (introspection, autocomplete) |
| প্রধান ঝুঁকি | Over/under-fetching, endpoint sprawl | N+1 queries, কঠিন caching/rate limiting |

## N+1 Problem বাস্তবে

- Query: 20 জন users এবং প্রতিটি user-এর orders নিয়ে আসা।
- Naive resolver approach: 20 জন users-এর জন্য 1টি query + তাদের orders-এর জন্য 20টি আলাদা query (প্রতি user-এর জন্য একটি) = মোট 21টি query।
- Batched (DataLoader) approach: 20 জন users-এর জন্য 1টি query + 1টি batched query ("এই 20টি user ID-র orders নিয়ে আসো") = মোট 2টি query।

## GraphQL কখন তার Complexity অর্জন করে

| Signal | কোনটি favor করে |
|---|---|
| ভিন্ন data প্রয়োজনসহ একাধিক independently-evolving clients (iOS, Android, web, TV) | GraphQL |
| একক client type, সহজ, স্থিতিশীল data প্রয়োজন | REST |
| HTTP/CDN caching-এর উপর ভারী নির্ভরতা | REST |
| ভিন্ন field combination প্রয়োজন হয় এমন ঘন ঘন screen/feature changes | GraphQL |
| সহজ, cacheable, well-understood semantics প্রয়োজন এমন external third parties-এর জন্য public API | REST |

## মূল সংখ্যা / তথ্য (Key Numbers / Facts)

- GraphQL 2012 সালে Facebook-এ internally develop করা শুরু হয়েছিল এবং 2015 সালে open-source করা হয়।
- একটি GraphQL schema introspectable — clients এবং tools (যেমন GraphiQL) docs generate করতে, autocomplete করতে, এবং type-checked queries তৈরি করতে schema-টিকেই query করতে পারে।

## সারসংক্ষেপ (Summary)

- REST-এর fixed-shape endpoints over-fetching এবং under-fetching তৈরি করে; GraphQL client-কে একটি endpoint এবং schema থেকে ঠিক যে data shape দরকার তা নির্দিষ্ট করতে দেয়।
- N+1 query problem হলো GraphQL-এর মূল operational ঝুঁকি, যা batching/DataLoader patterns দিয়ে সমাধান করা হয়।
- GraphQL flexibility-র বিনিময়ে REST-এর বিনামূল্যের HTTP caching এবং simple rate limiting ত্যাগ করে — এটি ঠিক তখনই worth it যখন একাধিক, ভিন্ন-আকৃতির client একটি shared backend-এর বিপরীতে evolve করে।
