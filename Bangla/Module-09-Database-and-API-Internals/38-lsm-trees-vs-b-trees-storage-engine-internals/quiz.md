# অনুশীলন ও ইন্টারভিউ প্রশ্ন (Practice & Interview Questions)

**১. B-tree write-এ কেন random disk I/O প্রয়োজন হয়, অথচ LSM tree write sequential হয়?**
একটি B-tree কোনো row-কে in place update করে, অর্থাৎ সেই row বর্তমানে যে page-এ আছে তা খুঁজে বের করে modify করা লাগে — সেই page যেকোনো জায়গায় থাকতে পারে, তাই write disk জুড়ে ছড়িয়ে পড়ে (random I/O)। একটি LSM tree বিদ্যমান data কখনোই in place modify করে না; প্রতিটি write একটি in-memory memtable এবং একটি write-ahead log-এর শেষে append করা হয়, যা কোন key লেখা হচ্ছে তা নির্বিশেষে সবসময় sequential operation।

**২. একটি memtable কী, এবং এটি পূর্ণ হয়ে গেলে কী ঘটে?**
memtable হলো in-memory, sorted structure, যা LSM tree-তে সাম্প্রতিক write buffer করে রাখে। এটি পূর্ণ হয়ে গেলে, disk-এ একটি নতুন immutable SSTable (Sorted String Table) হিসেবে flush করা হয়, এবং একটি নতুন, খালি memtable নতুন write গ্রহণ করা শুরু করে।

**৩. LSM tree-তে একটি single read কেন একাধিক SSTable check করা লাগতে পারে?**
কারণ update এমনকি delete-ও in-place modification হিসেবে নয়, বরং নতুন appended entry হিসেবে implement করা হয়, একই key-র entry memtable-এ এবং ভিন্ন সময়ে লেখা একাধিক SSTable-এ থাকতে পারে। একটি read-কে সেই key-র সবচেয়ে সাম্প্রতিক ভার্সন খুঁজে পেতে এগুলো (newest আগে) check করতে হয়।

**৪. Compaction কী, এবং এটি কেন প্রয়োজনীয়?**
Compaction একটি background process, যা একাধিক SSTable-কে কম সংখ্যক, বড়, sorted file-এ merge করে, updated key-র পুরনো ভার্সন এবং যেকোনো tombstoned (deleted) key বাদ দিয়ে। এটি না থাকলে, read-কে ক্রমবর্ধমান সংখ্যক SSTable check করতে হতো (read amplification বাড়ত) এবং disk usage stale data দিয়ে সীমাহীনভাবে বাড়ত (space amplification বাড়ত)।

**৫. Write amplification, read amplification, এবং space amplification সংজ্ঞায়িত করুন।**
Write amplification হলো অ্যাপ্লিকেশন যে logical byte লিখেছে তার তুলনায় disk-এ আসলে কত byte লেখা হয়েছে, তার অনুপাত। Read amplification হলো একটি logical read সন্তুষ্ট করতে কতগুলো disk read প্রয়োজন। Space amplification হলো data-র logical size-এর চেয়ে অতিরিক্ত কত disk space ব্যবহার হয়েছে। প্রতিটি storage engine এগুলোর মধ্যে trade-off করে।

**৬. Bloom filter কেন LSM tree read-কে সাহায্য করে, এবং এটি ব্যবহারের trade-off কী?**
একটি Bloom filter দ্রুত একজন reader-কে বলে দিতে পারে যে কোনো key একটি নির্দিষ্ট SSTable-এ *নিশ্চিতভাবে নেই*, ফলে read সম্পূর্ণভাবে সেই file এড়িয়ে যেতে পারে কোনো disk access ছাড়াই — যা read amplification কমায়। এর trade-off হলো Bloom filter-এর একটি ছোট false-positive rate থাকে (এটি এমন কোনো key-র জন্য "সম্ভবত আছে" বলতে পারে যা আসলে নেই, যার ফলে একটি অপ্রয়োজনীয় lookup হয়), যদিও এটি কখনো false negative তৈরি করে না।

**৭. পরিস্থিতি: আপনি এমন একটি system-এর জন্য storage engine বেছে নিচ্ছেন যা প্রতি সেকেন্ডে লক্ষ লক্ষ sensor reading ingest করে, এবং যেগুলো বেশিরভাগ সাম্প্রতিক data-র জন্য query করা হয়। কোন storage engine family বেশি উপযুক্ত, এবং কেন?**
একটি LSM-tree-ভিত্তিক engine (যেমন Cassandra বা RocksDB/LevelDB-এর উপর নির্মিত একটি time-series database) বেশি উপযুক্ত — workload-টি অত্যন্ত write-heavy এবং প্রায় সম্পূর্ণ-append, যা সরাসরি LSM tree-র শক্তির সাথে মিলে যায়: write-কে দ্রুত sequential I/O-তে রূপান্তর করা, এবং read pattern (বেশিরভাগ সাম্প্রতিক data) সাধারণত সদ্য flush হওয়া SSTable-গুলো hit করে, যেগুলোতে এখনো বেশি read amplification জমা হয়নি।

**৮. পরিস্থিতি: আপনি একটি e-commerce product catalog-এর জন্য storage engine বেছে নিচ্ছেন, যেখানে filtering, sorting, এবং range query ভারী, এবং write volume মাঝারি। কোনটি বেশি উপযুক্ত, এবং কেন?**
একটি B-tree-ভিত্তিক engine (যেমন PostgreSQL বা MySQL/InnoDB) বেশি উপযুক্ত — workload-টি read-heavy এবং range-query-heavy, যা ঠিক সেটাই যার জন্য B-tree optimize করা, এবং মাঝারি write volume B-tree-র in-place-update খরচের বিরুদ্ধে ততটা চাপ সৃষ্টি করে না যতটা একটি high-ingest workload করত।

**৯. একটি LSM-tree database-এ compaction অবহেলা করলে কেন শেষ পর্যন্ত read performance এবং disk usage উভয়েরই ক্ষতি হতে পারে?**
Compaction ছাড়া, SSTable ক্রমাগত জমা হতে থাকে এবং updated/deleted key-র পুরনো, প্রতিস্থাপিত ভার্সন কখনো পরিষ্কার হয় না। Read-কে ক্রমবর্ধমান সংখ্যক SSTable check করতে হয় (উচ্চ read amplification, ধীর read), এবং disk usage এমন stale data দিয়ে বাড়তে থাকে যা logically আর প্রয়োজন নেই (উচ্চ space amplification)।

**১০. সত্য অথবা মিথ্যা: একটি B-tree-ভিত্তিক database কখনোই উচ্চ write throughput সামলাতে পারে না।**
মিথ্যা, তবে একটি গুরুত্বপূর্ণ শর্ত সহ — B-tree যথেষ্ট উচ্চ write throughput সামলাতে পারে, এবং বাস্তব-জগতের B-tree database-গুলো write performance বাড়াতে নানা কৌশল ব্যবহার করে (write-ahead log, buffering, batching)। মূল বিষয়টি *আপেক্ষিক* — অত্যন্ত উচ্চ, sustained write volume-এর জন্য, একটি LSM-tree engine-এর append-only design মৌলিকভাবে সেই random I/O এবং page-splitting খরচ এড়িয়ে যায় যা B-tree প্রতিটি in-place update-এ দিতে হয়, এবং এই কারণেই সবচেয়ে উচ্চ-write-throughput system-গুলো সাধারণত LSM tree বেছে নেয়।
</content>
