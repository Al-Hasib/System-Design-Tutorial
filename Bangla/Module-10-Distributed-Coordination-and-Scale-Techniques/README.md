# Module 10: Distributed Coordination & Scale Techniques

Module 6-এ বড় নামের distributed systems সমস্যাগুলো covered হয়েছিল — consensus, distributed transaction, consistency model। এই module আরও তিনটা tool নিয়ে আসে যা আপনি আসলেই বড় scale-এ কিছু বানাতে গেলে বারবার সামনে আসে, কিন্তু খুব কমই আলাদাভাবে আলোচিত হয়: একাধিক independent process কীভাবে synchronize করার জন্য কোনো shared memory space ছাড়াই নিরাপদে একটা single resource share করে, একটা distributed system কীভাবে event-গুলো কোন *order*-এ ঘটেছে তা নিয়ে একমত হয় যখন নির্ভর করার মতো কোনো single shared clock নেই, এবং কীভাবে বিশাল dataset-এর উপর approximate প্রশ্নের উত্তর দেবেন ("আমি কি এটা আগে দেখেছি?", "মোটামুটি কতগুলো unique item?") একটা exact answer-এর memory cost ছাড়াই। এগুলোর কোনোটাই exotic না — এগুলো real production system-এ বারবার দেখা যায়, এবং interview-এও যখনই কোনো সমস্যার scale একটা exact, naive সমাধানকে স্পষ্টতই অনেক ব্যয়বহুল করে তোলে।

## এই Module-এর Video সমূহ

| # | Title | Description | Link |
|---|-------|-------------|------|
| 40 | Distributed Locking: Redlock, ZooKeeper & etcd | একাধিক independent process কীভাবে নিরাপদে একটা shared resource-এ exclusive access coordinate করে — এবং কেন একটা single-node lock (যেমন database row lock) distributed হয়ে গেলে যথেষ্ট না। | [40-distributed-locking-redlock-zookeeper-and-etcd](40-distributed-locking-redlock-zookeeper-and-etcd/README.md) |
| 41 | Logical Clocks & Time in Distributed Systems | কেন আপনি machine জুড়ে event order করতে wall-clock time-কে বিশ্বাস করতে পারেন না, এবং Lamport timestamp ও vector clock কীভাবে এর বদলে একটা নির্ভরযোগ্য "happened-before" সম্পর্ক প্রতিষ্ঠা করে। | [41-logical-clocks-and-time-in-distributed-systems](41-logical-clocks-and-time-in-distributed-systems/README.md) |
| 42 | Probabilistic Data Structures: Bloom Filters, HyperLogLog & Count-Min Sketch | বিশাল dataset-এর উপর একটা ছোট, fixed পরিমাণ memory ব্যবহার করে "আমি কি এটা দেখেছি?", "মোটামুটি কতগুলো unique?", এবং "মোটামুটি কতবার?" প্রশ্নের উত্তর দেওয়া। | [42-probabilistic-data-structures-bloom-filters-hyperloglog-and-count-min-sketch](42-probabilistic-data-structures-bloom-filters-hyperloglog-and-count-min-sketch/README.md) |
