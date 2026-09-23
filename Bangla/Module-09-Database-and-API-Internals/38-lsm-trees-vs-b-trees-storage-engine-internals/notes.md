# Study Notes: LSM Trees vs. B-Trees

## সংজ্ঞা (Definitions)

- **B-tree:** fixed-size page-এর একটি balanced tree, যা sorted key সংরক্ষণ করে; update in place ঘটে, যার জন্য সংশ্লিষ্ট page-এ seek করা প্রয়োজন।
- **LSM (Log-Structured Merge) tree:** একটি storage structure, যেখানে write একটি in-memory memtable এবং একটি write-ahead log-এ append করা হয়, তারপর immutable, sorted SSTable হিসেবে flush করা হয় — কখনোই in place modify করা হয় না।
- **Memtable:** in-memory, sorted structure যা disk-এ flush হওয়ার আগে সাম্প্রতিক write buffer করে রাখে।
- **SSTable (Sorted String Table):** একটি immutable, sorted on-disk file, যা memtable flush হলে তৈরি হয়।
- **Write-ahead log (WAL):** একটি append-only log, যা memtable update-এর আগে/পাশাপাশি লেখা হয়, crash recovery-র জন্য।
- **Compaction:** একটি background process, যা একাধিক SSTable-কে কম সংখ্যক, বড় SSTable-এ merge করে, প্রতিস্থাপিত/deleted data বাদ দিয়ে।
- **Write amplification:** অ্যাপ্লিকেশন যে logical byte লিখেছে তার তুলনায় disk-এ আসলে কত byte লেখা হয়েছে, তার অনুপাত।
- **Read amplification:** একটি logical read সন্তুষ্ট করতে কতগুলো disk read প্রয়োজন।
- **Space amplification:** data-র logical size-এর চেয়ে অতিরিক্ত কত disk space ব্যবহার হয়েছে।

## B-Tree বনাম LSM Tree

| দিক | B-Tree | LSM Tree |
|---|---|---|
| Write pattern | In-place update (random I/O) | Append-only (sequential I/O) |
| Write throughput | মাঝারি | উচ্চ |
| Read pattern | সরাসরি page traversal (দ্রুত, predictable) | memtable + একাধিক SSTable check করতে হতে পারে |
| Read throughput | উচ্চ, predictable | ধীর হতে পারে, Bloom filter দিয়ে কমানো হয় |
| Range query | চমৎকার (data in place sorted) | ভালো, কিন্তু একাধিক SSTable জুড়ে merge করা লাগতে পারে |
| Background maintenance | মাঝেমধ্যে page split/rebalancing | চলমান compaction |
| সাধারণ ব্যবহারকারী | PostgreSQL, MySQL/InnoDB, বেশিরভাগ RDBMS | Cassandra, RocksDB, LevelDB, HBase, অনেক time-series DB |
| সবচেয়ে উপযুক্ত | Read-heavy, range-query-heavy workload | Write-heavy, append-heavy, high-ingest workload |

## Amplification Trade-off Triangle

| Engine | Write amplification | Read amplification | Space amplification |
|---|---|---|---|
| B-tree | মাঝারি-উচ্চ (page rewrite/split) | কম | কম |
| LSM tree (compaction হয়নি) | Write path-এ কম | উচ্চ (অনেক SSTable check করতে হয়) | উচ্চ (পুরনো ভার্সন এখনো পরিষ্কার হয়নি) |
| LSM tree (compaction-এর পর) | বেশি (compaction-এর সময় data পুনরায় লেখা হয়) | কম (কম SSTable) | কম (প্রতিস্থাপিত data বাদ দেওয়া হয়েছে) |

কোনো engine একসাথে তিনটিকেই minimize করে না — compaction strategy ঠিক করে দেয় একটি LSM tree এই triangle-এর কোথায় বসবে।

## LSM Tree Write এবং Read কীভাবে কাজ করে

1. **Write:** write-ahead log-এ append করা হয় (durability) → in-memory memtable-এ insert করা হয় (sorted)।
2. **Flush:** memtable পূর্ণ হয়ে গেলে, একটি নতুন immutable SSTable হিসেবে disk-এ flush করা হয়।
3. **Read:** প্রথমে memtable check করা হয়, তারপর SSTable-গুলো newest-to-oldest check করা হয় (Bloom filter সেসব SSTable বাদ দেয় যেখানে key নিশ্চিতভাবে নেই), এবং পাওয়া সবচেয়ে নতুন ভার্সনটি ফেরত দেওয়া হয়।
4. **Compaction:** পর্যায়ক্রমে একাধিক SSTable merge করা হয়, obsolete/deleted ভার্সন বাদ দিয়ে, কম সংখ্যক বড়, sorted file তৈরি করা হয়।

## গুরুত্বপূর্ণ সংখ্যা / তথ্য (Key Numbers / Facts)

- Sequential disk I/O random I/O-র চেয়ে এক order of magnitude (অথবা তারও বেশি) দ্রুত হতে পারে, বিশেষত spinning disk-এ — এটাই মূল কারণ কেন LSM tree append-only write-কে প্রাধান্য দেয়।
- Cassandra, RocksDB, LevelDB, এবং HBase — সবগুলোই LSM-tree storage engine-এর উপর নির্মিত।
- PostgreSQL এবং MySQL/InnoDB (দুটি সবচেয়ে বহুল ব্যবহৃত open-source relational database) উভয়ই তাদের primary storage structure হিসেবে B-tree index ব্যবহার করে।

## সারসংক্ষেপ (Summary)

- B-tree data-কে in place update করে দ্রুত, predictable read এবং range scan-এর জন্য optimize করে — বিনিময়ে random-write I/O-র খরচ বহন করে।
- LSM tree শুধু কখনোই append করে write-কে দ্রুত করার জন্য optimize করে — বিনিময়ে read-path জটিলতা (একাধিক SSTable) এবং চলমান compaction-এর প্রয়োজনীয়তার খরচ বহন করে।
- storage engine family-কে workload-এর প্রকৃত read/write ratio-র সাথে মেলান, কোনো একটিমাত্র সার্বজনীন "database" internal structure ধরে নেওয়ার পরিবর্তে।
</content>
