# Study Notes: Distributed Locking

## সংজ্ঞাসমূহ

- **Distributed lock:** এমন একটি mechanism যা নিশ্চিত করে যে ভিন্ন ভিন্ন machine-এ থাকা অনেক process-এর মধ্যে শুধু একজনই একবারে একটি resource-এর ওপর exclusive access ধরে রাখতে পারে।
- **Redlock:** একটি Redis-ভিত্তিক distributed locking algorithm যার জন্য প্রয়োজন independent Redis instance-গুলোর একটি majority (যেমন, ৫টির মধ্যে ৩টি) একটি time budget-এর মধ্যে একমত হোক যে একটি lock acquire হয়েছে।
- **Ephemeral node (ZooKeeper):** এমন একটি node যা তৈরি করা client-এর session মারা গেলে স্বয়ংক্রিয়ভাবে অদৃশ্য হয়ে যায় — শুধুমাত্র timeout-এর ওপর নির্ভর না করে একজন crash হওয়া lock holder detect করতে ব্যবহৃত হয়।
- **Fencing token:** প্রতিটি lock grant-এর সাথে issue করা একটি strictly increasing সংখ্যা; সুরক্ষিত resource ইতিমধ্যে দেখা কোনো token-এর চেয়ে পুরনো token বহনকারী যেকোনো action প্রত্যাখ্যান করে, এটি একজন বিলম্বিত/stale holder-কে ক্ষতি করা থেকে প্রতিরোধ করে।

## পদ্ধতিসমূহের তুলনা

| পদ্ধতি | ভিত্তি | Correctness guarantee | পরিচালনাগত খরচ | সবচেয়ে উপযুক্ত |
|---|---|---|---|---|
| Redlock | independent Redis instance-গুলোর majority + expiry | ব্যবহারিক, কিন্তু বিতর্কিত timing assumption (clock sync, process pause) | কম — আপনি হয়তো ইতিমধ্যে চালানো Redis পুনর্ব্যবহার করুন | Non-critical deduplication (যেমন, কম-ঝুঁকির duplicate job এড়ানো) |
| ZooKeeper / etcd | Consensus protocol (ZAB / Raft) | শক্তিশালী — দ্বিমত ছাড়াই minority node failure সহ্য করে | বেশি — চালানো/operate করার জন্য একটি dedicated coordination cluster | উচ্চ-ঝুঁকির exclusive access (যেমন, leader election, critical single-execution job) |

## কেন শুধু একটি Lock যথেষ্ট নয়: Fencing Token সমস্যা

1. Client A একটি lock acquire করে।
2. Client A acquire করার *পরে* কিন্তু কাজ করার *আগে* pause করে (GC, network delay, OS scheduling)।
3. Lock expire হয়ে যায়; Client B এটি acquire করে, তার কাজ করে, এটি release করে।
4. Client A resume করে, তার lock-টি stale হয়ে গেছে তা না জেনে, তবুও কাজ করে — সম্ভাব্য conflict/corruption।
5. **সমাধান:** প্রতিটি lock grant-এ একটি strictly increasing fencing token অন্তর্ভুক্ত থাকে। সুরক্ষিত resource (যেমন, storage system) ইতিমধ্যে গ্রহণ করা সর্বোচ্চ token-এর চেয়ে পুরনো token-সহ যেকোনো write প্রত্যাখ্যান করে — তাই Client A-এর stale action প্রকৃত প্রভাবের মুহূর্তে প্রত্যাখ্যাত হয়, যদিও এর lock-acquisition check মূলত সফল হয়েছিল।

## মূল সংখ্যা / তথ্য

- Redlock-এর reference implementation সাধারণত ৫টি independent Redis instance ব্যবহার করে এবং একটি bounded time window-এর মধ্যে একটি majority (৩টি)-তে acquisition প্রয়োজন।
- ZooKeeper ZAB (ZooKeeper Atomic Broadcast) consensus protocol ব্যবহার করে; etcd Raft ব্যবহার করে — উভয়ই Module 6-এর consensus video-তে ধারণাগতভাবে কভার করা হয়েছে।
- Martin Kleppmann-এর Redlock-এর ব্যাপকভাবে উদ্ধৃত সমালোচনা ("How to do distributed locking," 2016) যুক্তি দেয় যে এর safety এমন কিছু assumption-এর ওপর নির্ভর করে (bounded clock drift, দীর্ঘ process pause না থাকা) যা বাস্তব system-এ guaranteed নয়।

## সারসংক্ষেপ

- Distributed lock একটি in-process mutex-এর মতোই একই mutual-exclusion সমস্যার সমাধান করে, কিন্তু এদের অবশ্যই একটি shared, network-accessible store-এ থাকতে হবে এবং deadlock বা double-granting না করে unreachable holder সামলাতে হবে।
- Redlock দ্রুত এবং practical কিন্তু এর পরিচিত, বিতর্কিত দুর্বলতা রয়েছে; ZooKeeper/etcd বেশি পরিচালনাগত খরচে শক্তিশালী, consensus-backed correctness প্রদান করে।
- একটি lock একা এই guarantee দেয় না যে সুরক্ষিত action নিরাপদে ঘটবে — fencing token "lock ধরে রাখা" এবং "action আসলে conflict ছাড়াই ঘটেছে" এর মধ্যকার ফাঁকটি বন্ধ করে দেয়।
