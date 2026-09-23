# এই Topic-টি কেন গুরুত্বপূর্ণ: Consensus Algorithms (Paxos & Raft)

> **এক বাক্যে বললে:** প্রতিটি distributed system-এর শেষ পর্যন্ত একগুচ্ছ machine-কে একটি একক value-তে একমত হতে হয়—কে leader, config-এ কী লেখা আছে, একটি lock ধরে আছে কিনা—এবং যখন message হারিয়ে যেতে পারে, বিলম্বিত হতে পারে, বা ক্রম-বিচ্যুতভাবে পৌঁছাতে পারে, তখন "শুধু vote নিয়ে নাও" পদ্ধতি কাজ করে না।

## এই ধারণার আগে যে পৃথিবী ছিল

আপনার কাছে তিনটি database replica আছে এবং primary-টি সাড়া দেওয়া বন্ধ করে দিয়েছে। বাকি দুটিকে একটি নতুন primary বেছে নিতে হবে। কীভাবে?

সরল (naive) পদ্ধতিগুলো চেষ্টা করে দেখুন এবং প্রতিটি কীভাবে ব্যর্থ হয় তা দেখুন:

- **Timeout হলে নিজেকে promote করা।** দুটি replica-ই timeout হয়, দুটিই নিজেদের promote করে। এখন দুটি primary write গ্রহণ করে, তারা বিচ্যুত হয়, এবং network সুস্থ হওয়ার পর তাদের একত্র করার কোনো সঠিক উপায় থাকে না। এটাই **split-brain**, এবং এর মানে স্থায়ী data loss।
- **একজন coordinator-কে জিজ্ঞাসা করা।** এখন coordinator-টিই একক ব্যর্থতার বিন্দু (single point of failure)। আপনি সমস্যাটি সরিয়েছেন, সমাধান করেননি।
- **Majority vote।** এটি কাছাকাছি—কিন্তু যদি message হারিয়ে না গিয়ে বিলম্বিত হয়? একটি node vote দিতে পারে, ক্র্যাশ করতে পারে, ভুলে গিয়ে পুনরায় চালু হতে পারে, এবং আবার vote দিতে পারে। দুটি node-ই ভাবতে পারে যে তারা জিতেছে।

এটি কঠিন হওয়ার কারণ হলো, একটি asynchronous network-এ **আপনি একটি ক্র্যাশ হওয়া node আর একটি ধীরগতির node-এর মধ্যে পার্থক্য করতে পারবেন না।** যে node সাড়া দেয়নি, সে হয়তো মৃত, অথবা হয়তো সাড়া দিতে যাচ্ছে। যেকোনো protocol যা অন্যথা ধরে নেয়, তা ত্রুটিপূর্ণ, এবং সবচেয়ে খারাপ সময়ে production-এ তা ভেঙে পড়বে।

## এটি যে সমস্যাগুলোর সমাধান করে

### ১. Split-brain এবং এর ফলে যে স্থায়ী বিচ্যুতি ঘটে
**আপনি যা দেখেন:** একটি network partition সুস্থ হওয়ার পর, দুটি node পরস্পরবিরোধী write গ্রহণ করে ফেলেছে। Data অপূরণীয়ভাবে ভুল হয়ে গেছে।

**এটি কেন ঘটে:** Partition-এর উভয় পক্ষই বিশ্বাস করেছিল যে তারাই দায়িত্বে আছে।

**Consensus কীভাবে এটি সমাধান করে:** একজন leader কেবল একটি **majority quorum** (অর্ধেকের বেশি node) দিয়ে নির্বাচিত হয়। যেহেতু একই সেটের দুটি majority-কে অন্তত একটি node-এ ওভারল্যাপ করতেই হবে, এবং সেই node একই term-এ দুবার vote দেবে না, তাই দুজন leader একই সময়ে নির্বাচিত হতে পারে না। সংখ্যালঘু (minority) পক্ষ একজন leader নির্বাচিত করতে *পারে না* এবং তাই write গ্রহণ করতে পারে না। Split-brain প্রতিরোধ করা হয় গণিত দিয়ে, আশা দিয়ে নয়।

### ২. Configuration নিয়ে বিভিন্ন node-এর মধ্যে মতবিরোধ
**আপনি যা দেখেন:** আপনার fleet-এর অর্ধেকে পুরনো shard map আছে, বাকি অর্ধেকে নতুনটি আছে, এবং request গুলো ভুল জায়গায় route হচ্ছে।

**এটি কেন ঘটে:** node-গুলোর কাছে পৃথকভাবে config push করে distribute করার কোনো atomicity নেই—কিছু node এটি পায়, কিছু পায় না, কিছু দেরিতে পায়।

**Consensus কীভাবে এটি সমাধান করে:** একটি replicated state machine। সব node একটি consensus-managed log থেকে একই command গুলো একই ক্রমে প্রয়োগ করে, তাই প্রতিটি node-এর state একই থাকে। ZooKeeper এবং etcd ঠিক এটাই—ছোট, অত্যন্ত নির্ভরযোগ্য, consensus-ভিত্তিক store যেখানে এমন facts থাকে যা নিয়ে সবার একমত হতে হবে। এই কারণেই Kubernetes তার সম্পূর্ণ cluster state etcd-তে সংরক্ষণ করে।

### ৩. Lock এবং lease যা আসলে নিরাপদ নয়
**আপনি যা দেখেন:** দুজন worker উভয়েই বিশ্বাস করে যে তারা lock ধরে আছে এবং দুজনেই একই job process করে, একজন customer-কে দুবার charge করে ফেলে।

**এটি কেন ঘটে:** একটি non-consensus store-এ (ধরুন, একটি একক Redis node) থাকা lock failover-এর সময় হারিয়ে যেতে পারে, অথবা একটি partition-এর সময় দুবার দেওয়া হতে পারে।

**Consensus কীভাবে এটি সমাধান করে:** একটি consensus-ভিত্তিক lock service একটি প্রকৃত guarantee দেয়, সেই সাথে fencing token—monotonically increasing সংখ্যা যা downstream system-গুলোকে পুরনো (stale) lock holder-এর একটি request প্রত্যাখ্যান করতে দেয়।

### ৪. একজনকে নির্বাচিত করার কোনো উপায় ছাড়াই "শুধু একজন leader ব্যবহার করো"
**আপনি যা দেখেন:** এমন architecture যা একজন একক writer, coordinator, বা scheduler ধরে নেয়, কিন্তু সে মারা গেলে কী হবে তার কোনো সংজ্ঞায়িত উত্তর নেই।

**এটি কেন ঘটে:** Leader-ভিত্তিক design গুলো নিয়ে যুক্তি দেওয়া অনেক সহজ, তাই সেগুলো বেছে নেওয়া হয়—এবং election সমস্যাটি পিছিয়ে দেওয়া হয়।

**Consensus কীভাবে এটি সমাধান করে:** বিশেষভাবে Raft leader election-কে একটি first-class, বোধগম্য mechanism-এ পরিণত করে (term, election timeout, heartbeat)। এই কারণেই Raft বাস্তবে Paxos-কে সরিয়ে দিয়েছে: Paxos সঠিক কিন্তু কুখ্যাতভাবে বোঝা কঠিন এবং সঠিকভাবে implement করা তার চেয়েও কঠিন, যেখানে Raft স্পষ্টভাবে বোধগম্যতার জন্য ডিজাইন করা হয়েছিল। সেই design লক্ষ্যটি নিজেই একটি engineering শিক্ষা।

## যে মূল্য আপনাকে দিতে হয় (The Price You Pay)

Consensus ব্যয়বহুল, ঠিক এই কারণেই আপনি এটি সীমিতভাবে ব্যবহার করেন:

- **প্রতিটি write-এর একটি majority-র কাছে round trip-এর খরচ হয়।** Latency আপনার সবচেয়ে ধীরগতির majority সদস্য দ্বারা নিচ থেকে সীমাবদ্ধ, যা region জুড়ে প্রতি operation-এ কয়েক দশ বা কয়েকশ মিলিসেকেন্ড মানে দাঁড়ায়। এটি একটি high-throughput data path নয়।
- **Throughput node-এর সাথে scale করে না—বরং খারাপ হয়।** বেশি node মানে প্রতিটি সিদ্ধান্তের জন্য বেশি message। পাঁচটি node সাধারণত সবচেয়ে উপযুক্ত সংখ্যা; নয়টি সাধারণত পাঁচটির চেয়ে খারাপ।
- **এটি নকশাগতভাবে availability বিসর্জন দেয়।** একটি majority প্রয়োজনীয়তার সাথে, majority হারানো মানে system সম্পূর্ণভাবে write গ্রহণ করা বন্ধ করে দেয়। এটাই সঠিক আচরণ (এটি CAP-এ CP পছন্দ), এবং এর মানে হলো একটি consensus system অসামঞ্জস্যের ঝুঁকি নেওয়ার চেয়ে ইচ্ছাকৃতভাবে বন্ধ হয়ে যাবে।
- **নিজে এটি implement করা একটি খারাপ ধারণা।** সঠিক consensus implementation পরিপক্ব হতে বছরের পর বছর সময় লাগে। etcd, ZooKeeper, Consul, অথবা এমন একটি database ব্যবহার করুন যাতে একটি ভালোভাবে পরীক্ষিত implementation embedded আছে।

**ব্যবহারিক নিয়ম:** ছোট পরিমাণের critical metadata-র জন্য consensus ব্যবহার করুন—leadership, membership, configuration, lock—এবং আপনার bulk data path-কে এর বাইরে রাখুন।

## কখন আপনার এটি প্রয়োজন — এবং কখন নয়

| আপনার consensus প্রয়োজন যখন | আপনার প্রয়োজন নেই যখন |
|---|---|
| ঠিক একটিমাত্র node-কে leader হতে হবে | Node গুলো stateless এবং পরিবর্তনযোগ্য (interchangeable) |
| Correctness, availability-র চেয়ে গুরুত্বপূর্ণ (financial, inventory) | Eventual consistency গ্রহণযোগ্য |
| Cluster membership বা config বৈশ্বিকভাবে একমত হতে হবে | দ্বন্দ্বগুলো পরবর্তীতে সমাধান করা যায় (CRDT, LWW) |
| আপনার প্রকৃতপক্ষে নিরাপদ distributed lock প্রয়োজন | একটি coordination service ইতিমধ্যে এটি আপনার জন্য সামলায় |

## Interview-এ এটি কেন দেখা যায়

Consensus একটি senior-level topic এবং interviewer রা এটি গভীরতা যাচাই করার জন্য ব্যবহার করেন। আপনাকে Raft implement করতে বলার সম্ভাবনা কম, কিন্তু আপনার এটি ব্যাখ্যা করতে সক্ষম হওয়া উচিত: কেন একটি সরল timeout-and-promote scheme split-brain সৃষ্টি করে, কেন একটি majority quorum দুজন leader প্রতিরোধ করে, কেন consensus মূল data path-এর জন্য খুব ধীর, এবং কোন বাস্তব system গুলো এটি ব্যবহার করে (etcd, ZooKeeper, Consul, Kafka-র controller, CockroachDB, Spanner)। এটা জানা যে consensus ইচ্ছাকৃতভাবে CP—যে এটি বিচ্যুত হওয়ার বদলে থেমে যাওয়া বেছে নেয়—এটি CAP-এর সাথে পরিষ্কারভাবে সংযুক্ত করে এবং প্রকৃত বোঝাপড়া হিসেবে ফুটে ওঠে।

## এটি কীভাবে সংযুক্ত

Consensus হলো **CAP** (topic 15)-এর CP কোণ, বাস্তব রূপে। এটিই যা **replication**-কে failover-এর সময় নিরাপদ করে তোলে, বিপজ্জনক নয় (topic 13), সঠিক **distributed locking**-এর ভিত্তি (topic 40), এবং cluster membership-এর পেছনের mechanism যার ওপর **consistent hashing** (topic 24) এবং **service discovery** (topic 31) নির্ভর করে। এটি **two-phase commit** (topic 28)-এর coordinator-এর নিচের যন্ত্রপাতিও, এবং **Kubernetes** (topic 44)-এর control plane।

**পরবর্তী:** [Distributed Transactions: 2PC & Saga](../28-distributed-transactions-2pc-and-saga/why.md) — একটি value নিয়ে নয়, বরং একটি multi-service operation ঘটেছে কিনা তা নিয়ে একমত হওয়া।
