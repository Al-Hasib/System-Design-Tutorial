# চিট শীট: একটি নিউজ ফিড সিস্টেম ডিজাইন করা (Twitter/Facebook)

ইন্টারভিউ রিভিউয়ের জন্য দ্রুত-রেফারেন্স নোট। `README.md` (পুরো script), `diagrams.md`, `quiz.md`, এবং `resources.md`-এর সাথে জোড়ায় ব্যবহার করুন।

## Requirements

**Functional**
- Post তৈরি করা (post store write path)।
- Follow/unfollow (directed, asymmetric follow graph)।
- Home timeline: একজন ব্যবহারকারী যাদের follow করে তাদের সবার post।
- Ranked feed (recency + affinity + predicted engagement), বিশুদ্ধভাবে chronological নয়।
- Scope-এর বাইরে: comments, likes, DMs, ads।

**Non-functional**
- Timeline লোডের জন্য low read latency (reads >> writes)।
- Eventual consistency গ্রহণযোগ্য (কয়েক সেকেন্ডের বিলম্ব ঠিক আছে)।
- Celebrity post থেকে আসা উচ্চ write fan-out সাবলীলভাবে শোষণ করতে হবে।
- Strong consistency-এর চেয়ে availability প্রাধান্য পাবে (AP-ঘেঁষা)।

## Capacity সংখ্যা (এই সেশনের জন্য ধরে নেওয়া)

| Metric | মান |
|---|---|
| Daily active users (DAU) | ৩০ কোটি (300 million) |
| প্রতি ব্যবহারকারী গড় posts/day | ২ → ~৬০ কোটি posts/day |
| গড় posts/sec | ~৭,০০০ (গড়), ~২১,০০০ (peak, 3x) |
| প্রতি ব্যবহারকারী গড় followers | ৫০০ |
| Celebrity account (১ কোটি+ followers) | ~১০ লাখ account |
| সবচেয়ে বড় celebrity account | ৫-১০ কোটি followers |
| Feed reads/day | ~৩০০ কোটি (১০ ওপেন/user/day) |
| গড় reads/sec | ~৩৫,০০০ |
| গড় fan-out writes/sec (সাধারণ ব্যবহারকারী) | ~৩৫ লাখ (7,000 posts/sec x 500 followers) |
| একক celebrity post fan-out burst | ৫-১০ কোটি writes পর্যন্ত |
| Precomputed timeline cache size | ~২৪ TB (300M users x 800 entries x 100 bytes) |

## Architecture সংক্ষিপ্তসার

```
Client -> Load Balancer -> Post Service -> sharded Post Store (Module 3: sharding)
                                    \-> publish event -> Message Queue / Pub-Sub (Module 5, Kafka)
                                                              \-> Fan-out Service -> Follow-Graph Store
                                                                          \-> Ranking Service
                                                                          \-> Timeline Cache (Module 4: Redis, cache-aside)

Client -> Load Balancer -> Feed Service -> Timeline Cache (hit) -> hydrate posts -> rank/merge -> response
                                    \-> (cache miss / celebrity merge) -> Post Store + Follow-Graph Store
```

- Post Service: post যাচাই + persist করে, "new post" event emit করে।
- Fan-out Service: event consume করে, follower সংখ্যা অনুযায়ী push নাকি skip সিদ্ধান্ত নেয়।
- Ranking Service: fan-out সময়ে (rough) এবং/অথবা read সময়ে (fine) candidate স্কোর করে।
- Timeline Cache: Redis, consistent hashing (Module 6) দ্বারা sharded, প্রতি ব্যবহারকারীতে sorted sets।
- Follow-Graph Store: অপ্টিমাইজড graph reads (কে-কাকে-follow-করে), posts থেকে আলাদাভাবে sharded।

## মূল সিদ্ধান্ত ও Trade-off

| Strategy | Write cost | Read cost | সবচেয়ে উপযোগী | মূল ঝুঁকি |
|---|---|---|---|---|
| Fan-out-on-write (push) | উচ্চ — প্রতি post-এ প্রতি follower-এ একটি write | নিম্ন — একটি single cache read | সাধারণ ব্যবহারকারী (~৫০০ followers) | Celebrity post-এ বিশাল write burst হয় (hot key) |
| Fan-out-on-read (pull) | নিম্ন — প্রতি post-এ একটি write | উচ্চ — read সময়ে প্রতিটি followee-এর post merge করতে হয় | Celebrity account (লক্ষ লক্ষ followers) | যারা অনেককে follow করে তাদের জন্য ধীর reads |
| Hybrid (এই ডিজাইনে ব্যবহৃত) | সাধারণ ব্যবহারকারীর জন্য push, celebrity-র জন্য push স্কিপ | Read সময়ে celebrity-র post merge করা | Twitter/Facebook স্কেলের production সিস্টেম | read-path-এ যোগ হওয়া জটিলতা (দুটি merge source) |

## অন্যান্য মূল সিদ্ধান্ত

- **Consistent hashing** (Module 6) post-store shard এবং cache-cluster key — উভয়কেই সমানভাবে বিতরণ করে, স্কেল-আউটের সময় reshuffling কমিয়ে দেয়, এবং targeted replica দিয়ে hot key আইসোলেট করতে সাহায্য করে।
- **Cache-aside** (Module 4) precomputed timeline-এর জন্য: আগে cache পড়া, miss হলে source থেকে rebuild করা, cache পুনরায় পূরণ করা।
- **Pub-sub / message queue** (Module 5, Kafka) Post Service-কে Fan-out Service থেকে decouple করে, celebrity fan-out-এর bursty load শোষণ করে, এবং replay/retry সক্ষম করে।
- **Eventual consistency** এখানে একটি স্পষ্ট, গৃহীত trade-off — এখানে strong consistency-এর জন্য অতিরিক্ত ইঞ্জিনিয়ারিং করবেন না।
