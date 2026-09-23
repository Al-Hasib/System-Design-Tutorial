# Study Notes: Data Consistency Models ও Idempotency

## সংজ্ঞা

- **Consistency model**: একটি contract যা নির্দিষ্ট করে যে replicate করা বা concurrently write করা data read করার সময় একজন client কোন value/ordering observe করতে পারে।
- **Linearizability (strong consistency)**: প্রতিটি operation তার শুরু এবং শেষের মধ্যবর্তী কোনো এক মুহূর্তে তাৎক্ষণিকভাবে ঘটে বলে মনে হয়; সব client operation-এর একই total order observe করে, যা real time-এর সাথে মেলে।
- **Eventual consistency**: যদি নতুন কোনো write না হয়, তাহলে সব replica শেষপর্যন্ত একই value-তে converge হবে; convergence-এর সময় বা মধ্যবর্তী ordering-এর উপর কোনো guarantee নেই।
- **Causal consistency**: causally সম্পর্কিত operation (happens-before, যেমন একটি comment-এর reply) সব node একই order-এ দেখে; unrelated (concurrent) operation ভিন্ন ভিন্ন node-এ ভিন্ন ভিন্ন order-এ দেখা যেতে পারে।
- **Read-your-writes**: একটি client পরবর্তী read-এ সবসময় তার নিজের আগের write দেখে।
- **Monotonic reads**: পরপর read জুড়ে একটি client কখনো data-কে "সময়ে পিছনে যেতে" দেখে না।
- **Monotonic writes**: একটি client-এর write সেই order-এই apply হয় যে order-এ সে সেগুলো issue করেছিল।
- **Session consistency**: উপরের client-centric guarantee-গুলোর একটি বান্ডেল, একটি client-এর session-এ সীমাবদ্ধ।
- **Idempotency**: একটি operation idempotent হয় যদি এটি N বার সম্পাদন করার প্রভাব একবার সম্পাদন করার মতোই হয়।
- **Idempotency key**: একটি logical operation-এর জন্য client-generated unique ID, যা server একই request-এর retry সনাক্ত করতে এবং নিরাপদে handle করতে ব্যবহার করে।

## Consistency Model তুলনা

| Model | Guarantee | Availability/Latency Cost | উদাহরণ System |
|---|---|---|---|
| Linearizable (strong) | Total real-time order; read সবসময় সর্বশেষ write দেখে | High — leader/quorum/consensus দরকার; partition-এর সময় minority side-এর জন্য unavailable | ZooKeeper, etcd, Google Spanner |
| Causal | Happens-before order সংরক্ষিত; concurrent op reorder হতে পারে | Medium — causal metadata দরকার (vector clocks) কিন্তু global lock-step নয় | COPS, কিছু Cosmos DB configuration |
| Read-your-writes / Monotonic reads | শুধুমাত্র per-client session guarantee | Low-medium — sticky session বা per-client version tracking | অনেক social/consumer app |
| Eventual | শুধুমাত্র convergence, কোনো ordering/সময় guarantee নেই | সর্বনিম্ন — সব replica স্বাধীনভাবে available | DNS, Cassandra (default), DynamoDB (eventual mode), S3 (ঐতিহাসিকভাবে) |

## Idempotent বনাম Non-Idempotent Operation

| Operation | Idempotent? | কেন |
|---|---|---|
| `PUT /user/5 {name: "Sam"}` | হ্যাঁ | একই state বারবার set করলে একই result হয় |
| `DELETE /user/5` | হ্যাঁ | ইতিমধ্যে-deleted resource delete করা কিছু করে না |
| `POST /orders` (নতুন order তৈরি) | না | প্রতিটি call একটি নতুন resource তৈরি করে — retry duplicate তৈরি করে |
| `POST /accounts/5/increment-balance` | না | পুনরাবৃত্তি একাধিকবার amount যোগ করে |
| Idempotency key-সহ `POST /payments` | Idempotent বানানো হয়েছে | Server key দিয়ে deduplicate করে, নিরাপদে retry করা যায় |

## Idempotency Key Implementation পদ্ধতি

- Client প্রতিটি logical operation-এর জন্য (প্রতিটি network attempt-এর জন্য নয়) একটি UUID তৈরি করে এবং একটি header-এ (যেমন, `Idempotency-Key`) পাঠায়।
- Server, key-টি প্রথম দেখার সময়, request process করে এবং `{key -> result}` (একটি TTL সহ, যেমন, 24 ঘণ্টা) একটি দ্রুত store-এ (Redis, বা key-এর উপর unique constraint সহ একটি DB table) সংরক্ষণ করে।
- একই key দিয়ে retry হলে, server short-circuit করে: reprocess না করেই stored result ফেরত দেয়।
- একই সময়ে একই key এলে race condition এড়াতে atomic "check-and-set" (unique constraint / conditional write) ব্যবহার করুন।

## Interview Revision Bullets

- Consistency model-এর পছন্দ replication layer-এ করা CAP/PACELC trade-off-এর সরাসরি ফলাফল।
- Linearizability হলো সবচেয়ে শক্তিশালী এবং সবচেয়ে ব্যয়বহুল guarantee; এর জন্য coordination দরকার (leader election, consensus, বা quorum read/write)।
- Eventual consistency availability/latency সর্বোচ্চ করে কিন্তু conflict resolution (LWW, vector clocks, CRDTs)-কে application বা storage engine-এর উপর ঠেলে দেয়।
- Causal consistency এবং session guarantee (read-your-writes, monotonic reads) মিলিয়ে consumer application-এর জন্য একটি সাধারণ ব্যবহারিক মধ্যবর্তী সমাধান।
- Idempotency consistency-এর সাথে orthogonal: এটি retry-কে নিরাপদ করার ব্যাপার, data কতটা fresh সেটার ব্যাপার নয় — কিন্তু distributed system-এ এই দুটি নিয়মিত পারস্পরিক ক্রিয়া করে (যেমন, eventual consistency-এর অধীনে একটি write retry করা)।
- HTTP semantics: GET, PUT, DELETE specification অনুযায়ী idempotent; POST নয় — এই কারণেই টাকা বা side effect জড়িত mutating POST endpoint-এর explicit idempotency key দরকার।
