# নোট: Monolith vs Microservices

## সংজ্ঞা

- **Monolith**: একটি system যা একটিমাত্র unit/artifact হিসেবে তৈরি এবং deploy করা হয়। অভ্যন্তরীণভাবে layered বা modular হতে পারে, কিন্তু সব module একটি process (বা তার অভিন্ন replica) এবং সাধারণত একটি database share করে।
- **Modular monolith**: এমন একটি monolith যার অভ্যন্তরীণ module-গুলোর সুস্পষ্ট boundary এবং interface আছে (এবং প্রায়শই একটি database-এর মধ্যে পৃথক schema) কিন্তু তবুও একটিমাত্র deployable unit হিসেবে ship হয়। microservices-এর দিকে যাওয়ার একটি সাধারণ ধাপ।
- **Microservices**: একটি system যা স্বাধীনভাবে deployable service-এ বিভক্ত, প্রতিটি নিজস্ব data store-এর মালিক, network-এর মাধ্যমে যোগাযোগ করে (synchronous RPC/REST/gRPC বা asynchronous messaging)।
- **Strangler fig pattern**: একটি ক্রমবর্ধমান migration কৌশল — একটি monolith-এর সামনে একটি facade/gateway বসানো এবং ধীরে ধীরে functionality-র অংশগুলো নতুন service দিয়ে প্রতিস্থাপন করা, ধাপে ধাপে traffic redirect করে, একটি ঝুঁকিপূর্ণ rewrite-এর পরিবর্তে।
- **Conway's Law**: System গুলো প্রায়শই তাদের নির্মাণকারী organization-এর যোগাযোগ কাঠামোকে প্রতিফলিত করে। Microservices প্রায়শই একটি technical tool-এর চেয়ে যতটা না, ততটাই একটি org-design tool।

## তুলনা তালিকা

| মাত্রা (Dimension) | Monolith | Microservices |
|---|---|---|
| Deployment unit | একটি একক artifact/process | অনেকগুলো স্বাধীন service |
| Data ownership | সাধারণত একটি shared database | প্রতিটি service নিজস্ব data store-এর মালিক |
| Transactions | সহজ ACID transaction | Distributed transaction (saga, 2PC) দরকার |
| Scaling | পুরো app একসাথে scale হয় | প্রতিটি service আলাদাভাবে scale হয় |
| Fault isolation | একটি crash সবকিছু বন্ধ করে দিতে পারে | একটি ব্যর্থ service আলাদা করা যায় (circuit breaker সহ) |
| Team autonomy | একটি codebase জুড়ে সমন্বয় দরকার | Team গুলো স্বাধীনভাবে service-এর মালিক হতে ও deploy করতে পারে |
| Technology choice | সাধারণত সবকিছুর জন্য একটি stack | Polyglot — প্রতি service-এ ভিন্ন ভিন্ন language/DB |
| Operational জটিলতা | কম (একটি pipeline, একগুচ্ছ log) | বেশি (service discovery, distributed tracing, orchestration) |
| Testing | সহজ end-to-end test | কঠিন — contract test, integration environment দরকার |
| Debugging | একটি stack trace, একটি log stream | Distributed tracing/correlation ID দরকার |
| উপযুক্ত ক্ষেত্র | ছোট-থেকে-মাঝারি team, প্রাথমিক পর্যায়ের product, strong consistency-র প্রয়োজনীয়তা | বড় organization, স্বাধীনভাবে scale হওয়া component, অনেক স্বায়ত্তশাসিত team |

## কখন কোনটি বেছে নেবেন

**Monolith / modular monolith প্রাধান্য দিন যখন:**
- Team ছোট (মোটামুটি একটি থেকে কয়েকটি team)।
- Product/domain boundary এখনও অস্পষ্ট (প্রাথমিক পর্যায়ের startup)।
- বেশিরভাগ operation জুড়ে strong transactional consistency দরকার।
- আপনি operational/infra overhead কমাতে চান।

**Microservices প্রাধান্য দিন যখন:**
- একাধিক team-এর একে অপরকে না আটকিয়ে স্বাধীনভাবে deploy করা দরকার।
- ভিন্ন ভিন্ন component-এর scaling বা resource profile খুবই ভিন্ন।
- Availability-র জন্য component-গুলোর মধ্যে failure isolation দরকার।
- প্রতিটি service-এর জন্য CI/CD, observability, এবং service discovery-র platform পরিপক্কতা আপনার আছে (বা তৈরি করতে ইচ্ছুক)।

## সংক্ষিপ্ত সারাংশ

- একটি monolith-এর নির্ধারক বৈশিষ্ট্য হলো deployment boundary, অভ্যন্তরীণ code quality নয়।
- Microservices in-process function call-কে network call-এ রূপান্তরিত করে — এর সাথে থাকা সব failure mode (latency, partial failure, timeout) সহ।
- Microservices জুড়ে distributed transaction-এর জন্য native ACID-এর বদলে Saga বা 2PC-এর মতো pattern দরকার (Module 6 দেখুন)।
- ডিফল্ট হিসেবে একটি modular monolith দিয়ে শুরু করুন; সুনির্দিষ্ট scaling/organizational সংকেত দেখা দিলে strangler fig pattern-এর মাধ্যমে migrate করুন।
- Segment-এর 2018 সালের "goodbye microservices" post-mortem হলো মিলে যাওয়া সমস্যা ছাড়া microservices গ্রহণের ক্লাসিক সতর্কতামূলক গল্প।
</content>
