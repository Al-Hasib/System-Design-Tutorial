# Consensus Algorithms: Paxos & Raft

**কঠিনতা:** Advanced

## শিখনের লক্ষ্যসমূহ (Learning Objectives)

- distributed consensus মৌলিকভাবে কেন কঠিন, তা ব্যাখ্যা করা, যার মধ্যে FLP impossibility result-এর অন্তর্নিহিত ধারণাও রয়েছে।
- Paxos-এর role গুলো (Proposer, Acceptor, Learner) এবং এর দুটি phase (Prepare/Promise, Accept/Accepted) বর্ণনা করা, এবং কেন একটি majority quorum safety নিশ্চিত করে তা ব্যাখ্যা করা।
- Raft-এর role গুলো (Leader, Follower, Candidate), term, leader election, এবং log replication বর্ণনা করা, যার মধ্যে commit rule-ও অন্তর্ভুক্ত।
- understandability, structure, এবং real-world adoption-এর দিক থেকে Paxos ও Raft-এর তুলনা করা।
- কোন production system কোন algorithm (বা তার কোনো variant) ব্যবহার করে এবং কেন, তা চিহ্নিত করা।

## স্ক্রিপ্ট (Script)

### শুরু / ভূমিকা (Hook / Intro)

কল্পনা করুন, পাঁচজন মানুষ দূর থেকে একটি কোম্পানি চালাচ্ছেন, এবং তাদের মধ্যকার ফোন লাইনগুলোর ওপর তারা সম্পূর্ণভাবে ভরসা করতে পারছেন না। কল কেটে যায়। কেউ হয়তো কথার মাঝখানে থেমে যায় এবং আর কখনো কথা বলে না। দুজন মানুষ হয়তো একই সময়ে ভাবতে পারে যে তারাই দায়িত্বে আছে। তবুও, এই দলটিকে একটি একক সিদ্ধান্তে—নির্ভরযোগ্যভাবে, সর্বসম্মতভাবে এবং অপরিবর্তনীয়ভাবে—একমত হতে হবে, যেমন "কে CEO" বা "লেজারের পরবর্তী লাইন আইটেম কী।" এটাই হলো distributed consensus, এবং এটি computer science-এর সবচেয়ে গভীর সমস্যাগুলোর একটি। আজ আমরা এমন দুটি algorithm নিয়ে আলোচনা করব যা বাস্তবে এই সমস্যার সমাধান করে: Paxos, যা মূল এবং কুখ্যাতভাবে কঠিন সমাধান, এবং Raft, যা বিশেষভাবে বোঝার সুবিধার জন্য ডিজাইন করা algorithm। সতর্কবার্তা—এটি এই পুরো কোর্সের সবচেয়ে কঠিন topic গুলোর একটি। আমরা এটিকে অতিরিক্ত সরলীকরণ করব না। চলুন শুরু করা যাক।

### Consensus কেন কঠিন

Consensus মানে হলো একগুচ্ছ distributed node-কে একটি একক value-তে একমত করানো, এমনকি যখন কিছু node ব্যর্থ হয় বা message বিলম্বিত, হারিয়ে যায়, বা ক্রম-বিচ্যুত হয়। শুনতে সহজ মনে হয়। কিন্তু তা নয়। ১৯৮৫ সালে, Fischer, Lynch, এবং Paterson এমন একটি ফলাফল প্রমাণ করেন যা এখন FLP impossibility নামে পরিচিত: একটি সম্পূর্ণ asynchronous network-এ—যেখানে message delay-এর কোনো সীমা নেই—কোনো deterministic consensus algorithm safety এবং termination উভয়টিই একসাথে নিশ্চিত করতে পারে না, যদি একটিমাত্র node-ও ব্যর্থ হতে পারে। সহজ ভাষায় বললে: আপনি এমন কোনো protocol তৈরি করতে পারবেন না যা সবসময় সঠিক এবং সবসময় শেষ হয়, যখন আপনি "একটি node ধীরগতির" আর "একটি node মৃত"—এই দুইয়ের পার্থক্য বুঝতে পারেন না।

বাস্তব system গুলো ধারণাগুলো শিথিল করে এই সমস্যা এড়িয়ে যায়—Paxos এবং Raft বাস্তবে ধরে নেয় যে network "partially synchronous", অর্থাৎ এটি বেশিরভাগ সময় মোটামুটি নির্ভরযোগ্য থাকে, এবং তারা timeout ব্যবহার করে progress নিশ্চিত করে, যদিও তাত্ত্বিকভাবে একটি প্রতিকূল (adversarial) network তাদের চিরকালের জন্য আটকে রাখতে পারে। এটি একটি গ্রহণযোগ্য trade-off, কারণ বাস্তব network গুলো চিরকাল প্রতিকূল থাকে না।

তারপর আছে failure model নিজেই। Node ক্র্যাশ করে। Network partition হয়—একটি switch নষ্ট হয়ে যায়, এবং আপনার পাঁচ-node cluster ভেঙে তিনটি node-এর একটি group ও দুটি node-এর একটি group-এ পরিণত হয়, যেখানে প্রতিটি পক্ষ নিজেদের মধ্যে কথা বলতে পারে কিন্তু বিভাজনের ওপারে পারে না। যদি উভয় পক্ষ স্বাধীনভাবে কাজ চালিয়ে যায় এবং write গ্রহণ করতে থাকে, তাহলে আপনি পাবেন split-brain: একই system-এর দুটি উপসেট, প্রত্যেকে বিশ্বাস করছে যে সে-ই কর্তৃত্বপূর্ণ, এবং অসামঞ্জস্যপূর্ণ state-এ বিচ্যুত হচ্ছে। Consensus algorithm গুলো বিশেষভাবে split-brain প্রতিরোধ করার জন্য তৈরি, কারণ এগুলো সিদ্ধান্তের জন্য একটি majority quorum—অর্ধেকের বেশি node—থাকার শর্ত দেয়, যাতে নির্দিষ্ট আকারের একটি cluster-এ দুটি বিচ্ছিন্ন majority কখনোই একই সময়ে বিদ্যমান থাকতে না পারে।

### Paxos — মূল সমাধান

Paxos-এর বর্ণনা ১৯৮০-এর দশকের শেষদিকে Leslie Lamport দিয়েছিলেন, যা ১৯৯৮ সালে প্রকাশিত হয়, এবং এটিই মৌলিক প্রমাণ যে FLP সত্ত্বেও, majority quorum এবং পর্যাপ্ত eventual synchrony থাকলে consensus অর্জনযোগ্য।

Paxos তিনটি role সংজ্ঞায়িত করে। Proposer একমত হওয়ার জন্য একটি value প্রস্তাব করে। Acceptor প্রস্তাবগুলোর ওপর ভোট দেয় এবং system-এর persistent memory গঠন করে। Learner জানতে পারে কোন value নির্বাচিত (chosen) হয়েছে। বাস্তবে, একটি physical node প্রায়ই একাধিক role পালন করে।

Paxos দুটি phase-এ চলে, এবং phase-এর সংখ্যা সঠিকভাবে বোঝা গুরুত্বপূর্ণ, কারণ এটি প্রতিটি বাস্তব implementation-এ দেখা যায়। Phase 1a হলো Prepare: একজন Proposer একটি proposal number বেছে নেয়—যা unique এবং monotonically increasing—এবং Acceptor-দের একটি majority-র কাছে একটি Prepare request পাঠায়। Phase 1b হলো Promise: প্রতিটি Acceptor, যে আগে দেখা যেকোনো কিছুর চেয়ে বড় সংখ্যাযুক্ত একটি Prepare পায়, একটি Promise দিয়ে সাড়া দেয়, যার অর্থ "আমি এর চেয়ে ছোট সংখ্যার কোনো proposal accept করব না," এবং এতে সে ইতিমধ্যে accept করা সর্বোচ্চ-সংখ্যাযুক্ত proposal (যদি থাকে) অন্তর্ভুক্ত করে। Phase 2a হলো Accept: একবার Proposer majority-র কাছ থেকে Promise শুনলে, সে একটি value সহ Accept request পাঠায়—হয় তার নিজের value, অথবা, গুরুত্বপূর্ণভাবে, যেকোনো Acceptor-এর রিপোর্ট করা সর্বোচ্চ-সংখ্যাযুক্ত proposal-এর value, যাতে algorithm কখনো এমন একটি value overwrite না করে যা হয়তো ইতিমধ্যে chosen হয়ে গেছে। Phase 2b হলো Accepted: প্রতিটি Acceptor, যে ইতিমধ্যে তার চেয়ে বড় কিছু promise করেনি, accept করে, এবং একবার majority একই value accept করলে, সেই value চূড়ান্তভাবে chosen হয়—স্থায়ীভাবে।

কেন একটি majority quorum safety নিশ্চিত করে? কারণ একই node-এর সেট থেকে নেওয়া যেকোনো দুটি majority-কে অন্তত একটি node-এ ওভারল্যাপ করতেই হবে। সেই ওভারল্যাপিং Acceptor-ই হলো যে Phase 1b-তে পূর্বে accept করা value-টি বহন করে নিয়ে যায়, এবং এটিই একই instance-এর জন্য দুটি ভিন্ন value কখনো chosen হওয়া থেকে প্রতিরোধ করে।

সৎ কথা বলতে গেলে: Paxos, যেভাবে specify করা হয়েছে, তা বোঝা কুখ্যাতভাবে কঠিন এবং সঠিকভাবে implement করা তার চেয়েও কঠিন। Lamport-এর নিজের paper-টি এতটাই ঘন যে তিনি পরে "Paxos Made Simple" লিখেছিলেন শুধুমাত্র এটি আবার ব্যাখ্যা করার জন্য। Single-instance Paxos শুধুমাত্র একটি value নিয়ে একমত হয়; বাস্তব system-এর সিদ্ধান্তের একটি ধারাবাহিক log দরকার হয়, যে কারণে বাস্তব deployment গুলো Multi-Paxos ব্যবহার করে—একটি leader-এর মতো optimization, যা একবার একজন Proposer স্থিতিশীল হিসেবে প্রতিষ্ঠিত হয়ে গেলে instance-এর একটি স্রোতের জন্য Phase 1 বাদ দেয়, এবং message overhead অনেকাংশে কমিয়ে দেয়।

### Raft — বোঝার সুবিধার জন্য ডিজাইন করা

Raft ২০১৪ সালে Diego Ongaro এবং John Ousterhout দ্বারা একটি স্পষ্ট design লক্ষ্য নিয়ে প্রকাশিত হয়: ঠিক একই সমস্যা সমাধান করা যা Multi-Paxos সমাধান করে, কিন্তু এমনভাবে বোঝার সুবিধাযুক্ত হওয়া যাতে শেখানো এবং সঠিকভাবে implement করা যায়। তারা সফল হয়েছিল—Raft এখন নতুন system-এর জন্য ডিফল্ট পছন্দ।

Raft consensus-কে তিনটি স্পষ্টতর sub-problem-এ বিভক্ত করে: leader election, log replication, এবং safety। যেকোনো সময়ে node-গুলোর তিনটি role-এর একটি থাকে: Follower, Candidate, অথবা Leader। সময়কে term-এ ভাগ করা হয়—logical, monotonically increasing epoch, প্রতিটিতে সর্বোচ্চ একজন Leader থাকে।

Leader election এভাবে কাজ করে: প্রতিটি Follower-এর একটি randomized election timeout থাকে। যদি সেই timeout শেষ হওয়ার আগে সে কোনো Leader-এর কাছ থেকে heartbeat না শোনে, তাহলে সে একজন Candidate হয়ে যায়, term সংখ্যা বাড়ায়, নিজেকে ভোট দেয়, এবং অন্য সব node-এর কাছে vote অনুরোধ করে। যদি সে cluster-এর একটি majority-র কাছ থেকে vote পায়, তাহলে সে সেই term-এর জন্য Leader হয়ে যায় এবং আরও নির্বাচন দমন করতে নিয়মিত heartbeat পাঠাতে শুরু করে। Timeout গুলোর randomization হলো মূল কৌশল—এটি পরিসংখ্যানগতভাবে অসম্ভাব্য করে তোলে যে দুটি node একই সাথে candidate হয়ে যাবে এবং vote ভাগ হয়ে যাবে, তাই বাস্তবে election দ্রুত converge করে।

Log replication হলো যেভাবে Leader কাজ সম্পন্ন করে। Client গুলো Leader-এর কাছে command পাঠায়, যা প্রতিটি command-কে তার local log-এ একটি নতুন entry হিসেবে যুক্ত করে এবং তারপর সেই entry-টি সমান্তরালভাবে Follower-দের কাছে replicate করে। একবার Leader নিশ্চিত হলে যে node-গুলোর একটি majority—নিজেকে সহ—স্থায়ীভাবে entry-টি যুক্ত করেছে, তখন সে entry-টি commit করে, তার state machine-এ প্রয়োগ করে, এবং client-কে সাড়া দেয়। তারপর সে পরবর্তী heartbeat-এ Follower-দের নতুন commit index সম্পর্কে জানায়, যাতে তারাও এটি প্রয়োগ করতে পারে। যেসব Follower পিছিয়ে আছে, তাদেরকে Leader তার log জোরপূর্বক তাদের ওপর চাপিয়ে দিয়ে হালনাগাদ করে, যা পুরো cluster জুড়ে log সামঞ্জস্য নিশ্চিত করে।

Safety নিয়ে বলতে গেলে: Raft নিশ্চিত করে যে একজন candidate কেবল তখনই একটি election জিততে পারে যখন তার log অন্তত cluster-এর majority-র log-এর মতোই up to date—শেষ log entry-র term এবং index তুলনা করে—যা নিশ্চিত করে যে নতুন নির্বাচিত একজন Leader কখনো ইতিমধ্যে-committed entry গুলো overwrite করবে না। এটি Paxos যা proposal number-এর মাধ্যমে implicitly অর্জন করে, তার একটি অনেক বেশি স্পষ্ট, structurally enforced সংস্করণ।

### Paxos বনাম Raft

দুটি algorithm-ই একই সমস্যা সমাধান করে এবং উভয়ই safety-এর জন্য majority quorum-এর ওপর নির্ভর করে, তাই গাণিতিকভাবে তারা যা নিশ্চিত করতে পারে তাতে সমতুল্য। পার্থক্যগুলো structure এবং ergonomics-এ। Paxos symmetric এবং ডিফল্টভাবে leaderless, যা তাত্ত্বিকভাবে মার্জিত কিন্তু বিশাল ফাঁক রেখে যায়—যেমন leader election এবং log management—যা যে কেউ এটি implement করছে তার জন্য "engineering exercise" হিসেবে থেকে যায়। Raft leader election এবং একটি শক্তিশালী leader-ভিত্তিক log replication মডেল সরাসরি specification-এর মধ্যেই বেক করে দেয়, যা পুরো system-টিকে যুক্তি দেওয়া সহজ, পরীক্ষা করা সহজ, এবং প্রথমবারেই সঠিকভাবে করা সহজ করে তোলে। এই কারণেই Raft নতুন distributed system-এর জন্য পছন্দের choice হয়ে উঠেছে, যদিও Paxos আগে এসেছিল এবং প্রমাণ করেছিল যে এটি সম্ভব।

### বাস্তব-জগতের উদাহরণ (Real-World Example)

Paxos এবং এর variant গুলো কিছু অত্যন্ত বড় system চালায়। Google-এর Chubby lock service একটি Paxos-ভিত্তিক replicated state machine চালায়, এবং Google Spanner তার transaction log গুলো data center জুড়ে replicate করতে Paxos ব্যবহার করে। Apache ZooKeeper ব্যবহার করে ZAB—ZooKeeper Atomic Broadcast protocol—যা মূলত Paxos-এর মতো, state update-এর একটি sequence-এর primary-backup replication-এর জন্য customized। Raft-এর দিকে, etcd—Kubernetes-এর পেছনের coordination store—সরাসরি Raft implement করে, এবং এই কারণেই Kubernetes control-plane node ব্যর্থতার পরেও cluster state না হারিয়ে টিকে থাকতে পারে। HashiCorp Consul এবং CockroachDB, একটি distributed SQL database, তাদের replication layer-এর জন্যও Raft ব্যবহার করে, ঠিক এই কারণে যে Raft-এর স্বচ্ছতা তাদের team-দের জন্য এটিকে আত্মবিশ্বাসের সাথে implement এবং operate করা সম্ভব করে তুলেছিল।

### সংক্ষিপ্তসার (Recap)

Consensus একটি distributed system-কে ব্যর্থতা এবং network অনির্ভরযোগ্যতা সত্ত্বেও একটি value-তে একমত হতে দেয়, এবং FLP আমাদের বলে যে এটি পুরোপুরিভাবে করা তাত্ত্বিকভাবে অসম্ভব—তাই Paxos এবং Raft উভয়ই বাস্তবে এটি কার্যকর করতে majority quorum এবং timeout ব্যবহার করে। Paxos, Proposer, Acceptor, এবং Learner-দের সাথে Prepare/Promise এবং Accept/Accepted phase চালিয়ে, প্রথম প্রমাণ ছিল যে এটি অর্জনযোগ্য, কিন্তু এটি implement করা কঠিন। Raft একই guarantee-গুলোকে একটি স্পষ্ট Leader, term, randomized election timeout, এবং majority-acknowledged log replication-এর চারপাশে পুনর্গঠন করে, একটু তাত্ত্বিক মাধুর্যের বিনিময়ে অনেক বেশি ব্যবহারিক স্বচ্ছতা পাওয়ার জন্য—এই কারণেই এটি এখন etcd, Consul, এবং CockroachDB-কে সমর্থন করে।

### এরপর কী (What's Next)

Consensus replica জুড়ে একমত হওয়া operation-এর একটি একক log পাওয়া নিশ্চিত করে—কিন্তু যখন একটি একক business transaction-এর একাধিক স্বাধীন service বা database জুড়ে data আপডেট করতে হয়, তখন কী হয়? এটি একটি ভিন্ন coordination সমস্যা। পরবর্তী বার, আমরা কভার করব Distributed Transactions: Two-Phase Commit এবং Saga pattern।

## মূল বিষয়সমূহ (Key Takeaways)

- একটি সম্পূর্ণ asynchronous network-এ, একটিমাত্র faulty node থাকলেও, consensus প্রমাণযোগ্যভাবে কঠিন (FLP impossibility); বাস্তব system গুলো progress করতে partial synchrony এবং timeout-এর ওপর নির্ভর করে।
- Majority quorum হলো Paxos এবং Raft উভয়ের মূল safety mechanism: একই cluster-এর যেকোনো দুটি majority-কে অবশ্যই ওভারল্যাপ করতে হবে, যা split-brain এবং পরস্পরবিরোধী সিদ্ধান্ত প্রতিরোধ করে।
- Paxos Phase 1 (Prepare/Promise) এবং Phase 2 (Accept/Accepted) জুড়ে Proposer, Acceptor, এবং Learner ব্যবহার করে; এটি প্রমাণযোগ্যভাবে সঠিক কিন্তু কুখ্যাতভাবে implement করা কঠিন, তাই বাস্তব deployment গুলো Multi-Paxos ব্যবহার করে।
- Raft consensus-কে leader election, log replication, এবং safety-তে বিভক্ত করে, term এবং randomized election timeout ব্যবহার করে একজন একক Leader নির্বাচিত করে যিনি সব log write চালান।
- একটি Raft log entry কেবল তখনই commit হয় যখন node-গুলোর majority (Leader-সহ) এটি স্থায়ীভাবে যুক্ত করে ফেলেছে—একটি N-node cluster-এ, আপনার প্রয়োজন floor(N/2) + 1 টি node।
- Paxos এবং Raft তত্ত্বগতভাবে সমান শক্তিশালী; Raft ব্যাপক বাস্তব-জগতের গ্রহণযোগ্যতা পেয়েছে (etcd, Consul, CockroachDB) কারণ এর স্পষ্ট leader-ভিত্তিক design সঠিকভাবে implement এবং operate করা অনেক সহজ।
