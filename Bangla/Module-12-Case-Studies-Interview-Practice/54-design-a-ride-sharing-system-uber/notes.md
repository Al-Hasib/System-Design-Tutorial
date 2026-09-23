# Interview Cheat Sheet: Ride-Sharing System (Uber-এর মতো)

## প্রয়োজনীয়তা (Requirements)

**Functional**
- Rider একটি ride অনুরোধ করেন (pickup + destination)।
- Rider-কে কাছাকাছি একজন available driver-এর সাথে match করা।
- Approach এবং trip চলাকালীন driver-এর (এবং rider-এর) real-time GPS tracking।
- Fare calculation, surge pricing সহ।
- সম্পূর্ণ trip lifecycle management + সম্পন্ন হলে payment।

**Non-functional**
- Low-latency matching (সেকেন্ড, দশ সেকেন্ড নয়)।
- High availability (partial failure-এর মধ্যেও app ব্যবহারযোগ্য থাকতে হবে)।
- Geospatial scale (বিশ্বব্যাপী লক্ষ লক্ষ চলমান driver)।
- Trip state-এর জন্য strong consistency (কোনো double-booking নেই) এবং payment (কোনো double-charging নেই); location data eventually consistent হতে পারে।

## Capacity সংখ্যা (রেফারেন্স)

| মেট্রিক | ধারণা | ফলাফল |
|---|---|---|
| Daily active rider | 20M | — |
| Daily active driver | 5M (peak-এ 2M একই সাথে active) | — |
| Location update interval | প্রতি active driver-এ প্রতি ৪ সেকেন্ডে | 2M / 4 ≈ peak-এ **প্রতি সেকেন্ডে 500,000 location update** |
| Rides/day | 20M rider × 1.2 rides/day | ≈ 24M rides/day |
| Ride requests/sec | 24M / 86,400 সেকেন্ড গড়, ×3 peak factor | ≈ গড়ে 280/sec, **peak-এ ~800-1,000/sec** |
| Trip record size | ~1 KB metadata + ~11 KB GPS trace (225 ping × 50 বাইট) | ≈ 12 KB/trip |
| Trip history storage | 24M trip/day × 12 KB | ≈ **প্রতিদিন 290 GB**, ≈ **বছরে 100 TB** |

মূল কথা: location write (500K/sec) ride request-কে (~1K/sec) দুই ক্রম মাত্রা (order of magnitude) ছাড়িয়ে যায় — এই কারণেই location data-এর জন্য একটি purpose-built, high-throughput, availability-favored store দরকার, trip/payment record-এর মতো একই database নয়।

## Architecture সারসংক্ষেপ

Rider/Driver app → Load Balancer → API Gateway →
- **Location Service** — GPS ping গ্রহণ করে, একটি geospatial index (geohash/quadtree) বজায় রাখে, consistent hashing-এর মাধ্যমে sharded।
- **Matching Service** — কাছাকাছি available driver-দের জন্য Location Service-কে কোয়েরি করে, candidate rank করে, driver assign করে।
- **Trip Service** — trip state machine-এর মালিকানা রাখে (requested → assigned → arrived → in-progress → completed → paid); trip-কে একটি Saga হিসেবে চালায়।
- **Payment Service** — fare calculation, surge pricing, trip সম্পন্ন হলে idempotent charge।
- **Pub/Sub + WebSocket** — প্রতি trip topic-এ live location fan-out, persistent WebSocket connection-এর মাধ্যমে rider/driver app-এ push করা।

Datastore: live location-এর জন্য একটি geospatially-sharded in-memory store (যেমন Redis geo); trip + payment record-এর জন্য একটি strongly-consistent relational/transactional store; pub/sub এবং saga coordination-এর জন্য একটি message broker; ঐতিহাসিক GPS trace-এর জন্য object/cold storage।

## মূল সিদ্ধান্ত ও Trade-off

| সিদ্ধান্ত | বিকল্প | পছন্দ ও কারণ |
|---|---|---|
| Geospatial index structure | Geohash বনাম Quadtree | সরলতা, string-prefix locality, এবং consistent-hashing key হিসেবে সহজ ব্যবহারের জন্য **Geohash**; driver density খুবই অসম হলে এবং adaptive cell sizing অতিরিক্ত জটিলতার যোগ্য হলে **Quadtree**। |
| Server জুড়ে location data বিতরণ | Static range sharding বনাম Consistent hashing | Geohash prefix-এর উপর **Consistent hashing** — location node যোগ/অপসারণ করলে geo-cell-এর কেবল একটি ছোট অংশ পুনর্বিন্যস্ত হয়, rebalancing খরচ কমিয়ে। |
| Service জুড়ে trip + payment সমন্বয় | 2PC বনাম Saga | **Saga** — 2PC পুরো trip সময়কাল (মিনিট) জুড়ে cross-service lock ধরে রাখবে; Saga local transaction + compensating action ব্যবহার করে (যেমন driver reservation ছেড়ে দেওয়া, ব্যর্থ payment retry/flag করা) এবং partial failure-কে সুন্দরভাবে সহ্য করে। |
| Retry-তে double-charge প্রতিরোধ | কোনোটিই না বনাম Idempotency key | প্রতিটি payment charge call-এ trip ID থেকে উদ্ভূত **Idempotency key** — duplicate request দুইবার চার্জ করার বদলে মূল ফলাফল ফেরত দেয়। |
| Consistency model: location data | CP বনাম AP | **AP (availability-favored)** — এক সেকেন্ড পুরনো driver position ক্ষতিকর নয়; partition-এর অধীনেও system-কে responsive থাকতে হবে। |
| Consistency model: trip state / payment | CP বনাম AP | **CP (consistency-favored)** — double-booking/double-charging এড়াতে driver "reserved" flag এবং payment charge অবশ্যই strongly consistent হতে হবে। |
| Real-time client update | Polling বনাম Pub/Sub + WebSocket | **Pub/Sub + WebSocket** — push-based delivery polling overhead এড়ায় এবং প্রায় তাৎক্ষণিক ম্যাপ update দেয়। |
