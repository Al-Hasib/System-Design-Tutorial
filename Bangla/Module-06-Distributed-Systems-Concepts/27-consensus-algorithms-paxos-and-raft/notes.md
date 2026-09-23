# Study Notes: Consensus Algorithms (Paxos & Raft)

## মূল সংজ্ঞাসমূহ (Core Definitions)

- **Consensus**: একগুচ্ছ distributed node-কে একটি একক value বা value-এর একটি sequence-এ একমত করানোর সমস্যা, node failure, message loss, delay, বা reordering সত্ত্বেও।
- **Quorum**: একটি সিদ্ধান্তকে বৈধ বলে গণ্য করার জন্য যে ন্যূনতম সংখ্যক node-কে অংশগ্রহণ করতে হবে। একটি N-node cluster-এ একটি **majority quorum** হলো `floor(N/2) + 1` টি node। একই cluster থেকে নেওয়া যেকোনো দুটি majority quorum-কে অন্তত একটি common node শেয়ার করতেই হবে—এই ওভারল্যাপই safety নিশ্চিত করে।
- **Split-brain**: একটি failure mode যেখানে একটি network partition একটি cluster-এর দুটি (বা ততোধিক) বিচ্ছিন্ন উপসেটকে প্রত্যেকে বিশ্বাস করতে বাধ্য করে যে তারা কর্তৃত্বপূর্ণ এবং স্বাধীনভাবে write গ্রহণ করে, যার ফলে বিচ্যুত, অসামঞ্জস্যপূর্ণ state তৈরি হয়। Majority-quorum consensus এটি প্রতিরোধ করে কারণ একটি partition-এর কেবল একটি পক্ষই (সর্বোচ্চ) কখনো একটি majority ধারণ করতে পারে।
- **Leader election**: এমন প্রক্রিয়া যার মাধ্যমে একটি distributed cluster একটি নির্দিষ্ট সময়ের জন্য সিদ্ধান্ত/write পরিচালনার দায়িত্বে থাকা একজন একক coordinating node (Leader) নির্বাচন করে, সাধারণত যখন কোনো বর্তমান Leader জানা নেই বা পূর্ববর্তী Leader ব্যর্থ হয়েছে বলে ধরে নেওয়া হয় তখন এটি শুরু হয়।
- **Log replication**: এমন mechanism যার মাধ্যমে একজন Leader (বা Proposer) operation/entry-র একটি ক্রমবদ্ধ sequence অন্যান্য node-এর কাছে propagate করে, যাতে সব replica committed state change-এর একই sequence-এ converge করে।
- **Term / Epoch**: পুরনো (stale) leader/proposal সনাক্ত করতে ব্যবহৃত একটি monotonically increasing logical clock value। Raft একে "term" বলে (প্রতি term-এ সর্বোচ্চ একজন Leader)। Paxos একই প্রভাব অর্জন করে monotonically increasing, বৈশ্বিকভাবে unique **proposal number** দিয়ে।
- **FLP impossibility (1985, Fischer–Lynch–Paterson)**: message delay-এর কোনো সীমা ছাড়া একটি সম্পূর্ণ asynchronous network model-এ, একটিমাত্র node ব্যর্থ হতে পারলেও, কোনো deterministic consensus algorithm safety এবং termination (liveness) উভয়টিই একসাথে নিশ্চিত করতে পারে না। বাস্তব system গুলো partial synchrony ধারণা এবং timeout দিয়ে এটি এড়িয়ে যায়, তাত্ত্বিক guarantee-র বিনিময়ে ব্যবহারিক progress পায়।

## Paxos বনাম Raft — তুলনামূলক টেবিল

| দিক | Paxos | Raft |
|---|---|---|
| Role গুলো | Proposer, Acceptor, Learner | Leader, Follower, Candidate |
| Structure | Symmetric, ডিফল্টভাবে leaderless | স্পষ্টভাবে leader-ভিত্তিক (strong leader) |
| Phase গুলো | Phase 1 Prepare/Promise, Phase 2 Accept/Accepted | Leader election (RequestVote), Log replication (AppendEntries) |
| Ordering primitive | Proposal number (unique, monotonic) | Term number + log index |
| বোঝার সুবিধা | কুখ্যাতভাবে কঠিন; মূল paper-এর একটি follow-up দরকার হয়েছিল ("Paxos Made Simple") | স্পষ্টভাবে বোঝার সুবিধার জন্য ডিজাইন করা (Ongaro & Ousterhout, 2014) |
| ব্যবহারিক variant | Multi-Paxos (একবার একটি স্থিতিশীল proposer/leader থাকলে instance-এর একটি স্রোত জুড়ে Phase 1 বাদ দেয়) | Raft নিজেই ইতিমধ্যে একটি multi-entry log protocol |
| Leader-ভিত্তিক? | স্বভাবতই নয়; Multi-Paxos efficiency-র জন্য একটি de facto leader যোগ করে | হ্যাঁ, design অনুযায়ী — সব write বর্তমান Leader-এর মধ্য দিয়ে যায় |
| Safety mechanism | Majority quorum overlap + সর্বোচ্চ-সংখ্যাযুক্ত accepted value বহন করে নিয়ে যাওয়া | Majority quorum overlap + "leader completeness" (candidate-এর log অবশ্যই একটি majority-র মতো up to date হতে হবে) |
| বাস্তব-জগতের গ্রহণযোগ্যতা | Google Chubby, Google Spanner (Paxos-ভিত্তিক log replication); ZooKeeper-এর ZAB Paxos-এর মতো | etcd (Kubernetes), HashiCorp Consul, CockroachDB |

## Raft-এর তিনটি Sub-Problem

- **Leader election** — প্রতি term-এ ঠিক একজন Leader, randomized election timeout এবং majority vote (RequestVote RPC)-এর মাধ্যমে নির্বাচিত।
- **Log replication** — Leader client command গুলো তার log-এ যুক্ত করে এবং Follower-দের কাছে replicate করে (AppendEntries RPC); একটি entry কমিট হয় একবার node-গুলোর majority এটি স্থায়ীভাবে সংরক্ষণ করলে।
- **Safety** — নিশ্চিত করে যে একবার একটি entry কমিট হয়ে গেলে, ভবিষ্যতের কোনো Leader তা overwrite বা হারাতে পারবে না; election restriction (একজন candidate-কে জিততে হলে অবশ্যই একটি up-to-date log থাকতে হবে) এবং commit rule (শুধুমাত্র বর্তমান term থেকে entry গুলো সরাসরি commit করে; আগের term গুলো transitively commit হয়) দিয়ে প্রয়োগ করা হয়।

## গুরুত্বপূর্ণ সংখ্যাসমূহ (Key Numbers)

- Majority = `floor(N/2) + 1`।
- 3-node cluster: majority = 2, 1টি node failure সহ্য করতে পারে।
- 5-node cluster: majority = 3, 2টি node failure সহ্য করতে পারে।
- 7-node cluster: majority = 4, 3টি node failure সহ্য করতে পারে।
- Fault tolerance না বাড়িয়ে বিজোড় সংখ্যার বাইরে node যোগ করা অপচয়মূলক—উদাহরণস্বরূপ, একটি 4-node cluster এখনও শুধু 1টি failure সহ্য করে (majority = 3) কিন্তু একই fault tolerance-এর একটি 3-node cluster-এর চেয়ে বেশি replication খরচ দেয়। এই কারণেই production Raft/Paxos cluster গুলো প্রায় সবসময় বিজোড় সংখ্যায় (3, 5, 7) সাইজ করা হয়।

## Interview পুনরালোচনা — দ্রুত পয়েন্টসমূহ

- Consensus = অনির্ভরযোগ্য node জুড়ে একটি value/log নিয়ে একমত হওয়া; Paxos এবং Raft উভয়ই safety-র জন্য majority quorum-এর ওপর নির্ভর করে, unanimity-র ওপর নয়।
- FLP impossibility বলে যে failure সহ একটি বিশুদ্ধ asynchronous model-এ নিখুঁত consensus (safety + নিশ্চিত termination) অসম্ভব—বাস্তব system গুলো এটি এড়াতে timeout এবং partial synchrony ব্যবহার করে।
- Paxos phase গুলো: Phase 1a Prepare / Phase 1b Promise, তারপর Phase 2a Accept / Phase 2b Accepted। একটি value chosen হয় একবার Acceptor-দের majority এটি accept করলে।
- Raft phase গুলো: Leader election (term, randomized timeout, vote) তারপর log replication (AppendEntries, majority ack-এ commit)।
- Split-brain প্রতিরোধ করা হয় কারণ একটি fixed-size cluster-এ দুটি বিচ্ছিন্ন majority একই সময়ে বিদ্যমান থাকতে পারে না।
- Raft raw Paxos-এর চেয়ে গ্রহণযোগ্যতা জিতেছে কারণ এটি সঠিকভাবে implement করা অনেক সহজ, তাত্ত্বিকভাবে বেশি শক্তিশালী বলে নয়—উভয়ই সমান শক্তিশালী।
- প্রতিটির জন্য একটি করে production উদাহরণ জানুন: etcd/Consul/CockroachDB → Raft; Chubby/Spanner → Paxos-variant; ZooKeeper → ZAB (Paxos-এর মতো)।
