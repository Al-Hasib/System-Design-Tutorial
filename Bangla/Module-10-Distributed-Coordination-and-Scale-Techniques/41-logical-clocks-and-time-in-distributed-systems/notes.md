# Study Notes: Logical Clocks & Time in Distributed Systems

## সংজ্ঞা (Definitions)

- **Clock skew/drift:** স্বাধীন মেশিনগুলোর physical clock-এর মধ্যে পার্থক্য/বিচ্যুতি, এমনকি NTP synchronization-এর অধীনেও।
- **NTP (Network Time Protocol):** reference time server-এর বিপরীতে মেশিন জুড়ে physical clocks সিঙ্ক্রোনাইজ করার protocol — ভালো অবস্থায় সাধারণত এক অঙ্কের মিলিসেকেন্ড পর্যন্ত সঠিক, কিন্তু নিশ্চিত নয়।
- **Lamport timestamp:** একটি per-process counter যা "happened-before" partial order প্রতিষ্ঠা করে: প্রতিটি local event-এ বৃদ্ধি পায়, এবং একটি message পাওয়ার সময় `max(own, received) + 1`-এ সেট হয়।
- **Happened-before relationship:** একটি partial order — যদি event A causally event B-কে প্রভাবিত করতে পারে, তাহলে A "happened-before" B।
- **Vector clock:** per-process counter-এর একটি array; প্রতিটি process locally শুধু তার নিজের slot বৃদ্ধি করে, এবং একটি message পাওয়ার সময় একটি element-wise max নেয় (তারপর নিজের slot বৃদ্ধি করে)।
- **Concurrent events:** দুটি events যেখানে কারোরই vector clock অন্যটিকে dominate করে না — কেউই causally অন্যটিকে প্রভাবিত করতে পারতো না।

## Lamport Timestamps বনাম Vector Clocks

|             | Lamport Timestamps                                                          | Vector Clocks                                                    |
| ----------- | --------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| গঠন (Structure)   | প্রতি process একটি একক counter                                                  | Counter-এর array, প্রতি process একটি                               |
| নিশ্চয়তা (Guarantees)  | যদি A happened-before B হয়, তাহলে timestamp(A) < timestamp(B)                    | happened-before নির্ধারণ করতে পারে এবং প্রকৃত concurrency শনাক্ত করতে পারে        |
| সীমাবদ্ধতা (Limitation)  | Concurrent (অসম্পর্কিত) events-কে causally ordered events থেকে আলাদা করতে পারে না | বেশি overhead — process সংখ্যার সাথে আকার বৃদ্ধি পায়           |
| সাধারণ ব্যবহার (Typical use) | মৌলিক event ordering, সাধারণ distributed logging                            | multi-replica systems-এ conflict detection (যেমন, Dynamo, Riak) |

## Vector Clock তুলনার নিয়ম (Comparison Rules)

Vector A এবং B দেওয়া থাকলে:

- **A happened-before B** যদি A-এর প্রতিটি element ≤ B-এর সংশ্লিষ্ট element হয়, এবং অন্তত একটি strictly ছোট হয়।
- **A এবং B concurrent** যদি কেউই অন্যটিকে dominate না করে (কিছু elements A-তে বেশি, অন্যগুলো B-তে বেশি) — এটিই একটি প্রকৃত conflict-এর সংকেত।

## কখন Physical Time বনাম Logical/Vector Clocks ব্যবহার করবেন

| ব্যবহারের ক্ষেত্র (Use case)                                                | সঠিক tool (Right tool)                                           |
| ------------------------------------------------------- | ---------------------------------------------------- |
| Human debugging-এর জন্য log timestamps                      | Physical (NTP) time                                  |
| Cache TTL / expiration windows                          | Physical (NTP) time — আনুমানিক precision যথেষ্ট |
| Rate-limiting windows                                   | Physical (NTP) time                                  |
| সঠিকতার জন্য causally-related events order করা        | Lamport timestamps                                   |
| Replica জুড়ে সাংঘর্ষিক concurrent writes শনাক্ত করা | Vector clocks                                        |

## মূল সংখ্যা / তথ্য (Key Numbers / Facts)

- Leslie Lamport-এর "Time, Clocks, and the Ordering of Events in a Distributed System" (1978) Lamport timestamps প্রবর্তন করে।
- NTP সাধারণত ভালো network অবস্থায় clocks-কে কয়েক মিলিসেকেন্ডের মধ্যে সিঙ্ক্রোনাইজ করে, কিন্তু এটি কোনো কঠোর guarantee নয় — network সমস্যা অনেক বড় skew তৈরি করতে পারে।
- Amazon-এর Dynamo পেপার (2007) eventually-consistent, multi-replica systems-এ conflict detection-এর জন্য vector clocks-কে জনপ্রিয় করে তোলে; "shopping cart merge" উদাহরণটি এই system থেকে এসেছে।

## সারসংক্ষেপ (Summary)

- মেশিন জুড়ে wall-clock time-কে সঠিকতা-নির্ভর event ordering-এর জন্য নির্ভরযোগ্য ধরা যায় না, এমনকি NTP থাকলেও।
- Lamport timestamp সস্তায় causal ("happened-before") order প্রতিষ্ঠা করে কিন্তু প্রকৃত concurrency-কে যথেচ্ছ ordering থেকে আলাদা করতে পারে না।
- Vector clocks প্রকৃত concurrency শনাক্ত করতে পারে, যা ঠিক তখন প্রয়োজন যখন জানতে হয় দুটি writes প্রকৃতপক্ষে conflict করছে এবং একটি নীরব, সম্ভাব্য ভুল timestamp-ভিত্তিক পছন্দের পরিবর্তে স্পষ্ট সমাধান প্রয়োজন।
</content>
