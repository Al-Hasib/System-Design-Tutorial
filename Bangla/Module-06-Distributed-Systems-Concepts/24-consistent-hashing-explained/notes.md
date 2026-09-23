# Study Notes: Consistent Hashing

## Definitions

- **Consistent hashing**: একটি hashing scheme যেখানে node এবং key উভয়কেই একই hash space-এ ম্যাপ করা হয়, এবং একটি node যোগ বা অপসারণের জন্য শুধুমাত্র key-এর একটি ছোট, সমানুপাতিক অংশ (প্রায় K/N) remap করার প্রয়োজন হয়, প্রায় পুরো key space remap করার বদলে।
- **Hash ring**: hash space-এর circular উপস্থাপনা (যেমন, 0 থেকে 2^32 - 1, maximum থেকে আবার 0-তে wrap করে), যার উপর node এবং key-এর hash value বসানো হয়। একটি key তার position থেকে clockwise দিকে হেঁটে পাওয়া প্রথম node-এর অন্তর্গত।
- **Virtual node (vnodes)**: প্রতি physical node-এ একাধিক hash position তৈরি করা হয় (যেমন, `node_id#1`, `node_id#2`, ... hash করে), যার প্রতিটি hash ring-এ স্বাধীনভাবে বসানো হয়। load distribution সমান করতে এবং node capacity অনুযায়ী weight করতে ব্যবহৃত হয়।
- **Routing consistency**: এই property যে independent client-রা, ring membership-এর একই view দেওয়া থাকলে, কোনো coordination ছাড়াই একই key-এর জন্য একই node গণনা করে। CAP-theorem "consistency" (replica জুড়ে data consistency) থেকে এটি ভিন্ন।

## Why Naive `hash(key) mod N` Fails at Scale

- Modulus N প্রতিটি key-এর assignment-এর মধ্যে বেক করা থাকে। N পরিবর্তন করলে (node যোগ/অপসারণ) প্রায় প্রতিটি key-এর `mod N` result পরিবর্তিত হয়, কারণ mod-N এবং mod-(N+1) assignment-এর মধ্যে কোনো structural সম্পর্ক নেই।
- সবচেয়ে খারাপ ক্ষেত্র: একটি একক node পরিবর্তনের জন্য সব key-এর `(N-1)/N` পর্যন্ত remap হয় (যেমন, একটি 10-node cluster-এর জন্য ~90%)।
- ফলাফল: ব্যাপক cache miss, বিশাল data-transfer/rebalancing খরচ, origin/backing store-এ সাময়িক overload, খারাপ incremental scalability।

## Comparison: Naive Hashing vs Consistent Hashing

| দিক | Naive `mod N` Hashing | Consistent Hashing (Ring) |
|---|---|---|
| Resize-এ remap হওয়া key | সব key-এর (N-1)/N পর্যন্ত (N=10-এর জন্য ~90%) | প্রায় K/N key (শুধু সংলগ্ন arc) |
| Rebalancing খরচ | বেশি — প্রায় সম্পূর্ণ data shuffle | কম — পরিবর্তনের আকারের সমানুপাতিক |
| Implementation জটিলতা | খুবই সহজ (একটি মাত্র modulo operation) | মাঝারি (ring structure, sorted position, successor দিয়ে lookup) |
| Node জুড়ে load balance | fixed N-এর জন্য, গঠনগতভাবেই সমান | প্রতি node-এ অল্প point থাকলে অসম; সমান করতে virtual node দরকার |
| Heterogeneous hardware সামলায় | না (custom logic দরকার) | হ্যাঁ — প্রতি node-এ virtual node সংখ্যা পরিবর্তন করে |
| প্রয়োজনীয় coordination | N নিয়ে global agreement দরকার | ring membership-এর eventually-consistent view দরকার (যেমন, gossip) |
| সাধারণ ব্যবহারের ক্ষেত্র | ছোট/fixed-size cluster, সহজ in-memory partitioning | Distributed cache, NoSQL store (DynamoDB, Cassandra, Riak), CDN/load-balancer routing |

## Key Numbers to Remember

- Naive resize disruption: key-এর `(N-1)/N` পর্যন্ত move করে (যেমন, N=10-এর জন্য 9/10 = 90%)।
- Consistent hashing resize disruption: প্রায় `K/N` key move করে (K = মোট key, N = node সংখ্যা)।
- Production system-এ প্রতি physical node-এ সাধারণ virtual node সংখ্যা: প্রায় 100-200 (কিছু সিস্টেম cluster size এবং কাঙ্ক্ষিত balance অনুযায়ী কম-বেশি configure করে)।
- Replication factor (R) ring-এর উপরে স্তরায়িত থাকে: write/read সাধারণত একটি key-এর position থেকে clockwise দিকে হেঁটে পরবর্তী R-টি ভিন্ন physical node স্পর্শ করে (সাধারণ default: Dynamo-স্টাইলের সিস্টেমে R = 3)।

## Interview Revision — Bullet Summary

- Naive mod-N hashing resize-এ ভেঙে পড়ে কারণ modulus পরিবর্তিত হয়; consistent hashing একটি ring এবং "clockwise-এ পরবর্তী node"-এর মালিকানা ব্যবহার করে এটি সমাধান করে।
- Node যোগ/অপসারণে শুধু topology পরিবর্তনের সংলগ্ন arc-ই প্রভাবিত হয় — প্রায় সব key-এর বদলে প্রায় K/N key move করে।
- Virtual node: প্রতি physical node-এ একাধিক ring position; (1) random placement থেকে আসা load imbalance, (2) heterogeneous capacity weighting, (3) failover load-কে একটি প্রতিবেশীর বদলে অনেক node জুড়ে কেন্দ্রীভূত করার সমস্যা সমাধান করে।
- "Consistent" = routing consistency (একই key, একই node, independent observer জুড়ে), CAP-স্টাইলের strong consistency নয়।
- বাস্তব সিস্টেম basic ring-এর উপরে replication (N replica), conflict resolution (vector clock, LWW, read-repair), এবং membership propagation (gossip) স্তরায়িত করে।
- বাস্তব-জগতের ব্যবহারকারী: DynamoDB, Cassandra (token ring + vnode), Riak, CDN, এবং sticky, resize-tolerant routing প্রয়োজন এমন load balancer/service mesh।
