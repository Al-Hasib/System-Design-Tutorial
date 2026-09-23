# অনুশীলন ও Interview প্রশ্নসমূহ (Practice & Interview Questions)

**১. একটি consensus algorithm আসলে কোন সমস্যার সমাধান করে, এবং কেন আপনি একটি formal protocol ছাড়া শুধু প্রতিটি node-কে "majority vote দিয়ে একমত হও" বলতে পারেন না?**
Consensus একগুচ্ছ distributed node-কে node failure এবং অনির্ভরযোগ্য message delivery সত্ত্বেও একটি একক value (বা value-এর একটি ক্রমবদ্ধ sequence) নিয়ে একমত করায়। একটি সরল majority vote ব্যর্থ হয় কারণ message বিলম্বিত, ক্রম-বিচ্যুত, বা হারিয়ে যেতে পারে, এবং node গুলো সিদ্ধান্তের মাঝপথে ব্যর্থ হতে পারে—একটি formal protocol ছাড়া (proposal number/term, quorum overlap নিয়ম, এবং সংজ্ঞায়িত phase), আপনি এমন অবস্থায় পৌঁছাতে পারেন যেখানে দুটি ভিন্ন value cluster-এর ভিন্ন অংশে "chosen" হিসেবে প্রতীয়মান হয়, যা ঠিক সেই safety violation যা consensus algorithm গুলো প্রতিরোধ করার জন্য তৈরি।

**২. FLP impossibility result আসলে কী বলে, এবং Paxos ও Raft বাস্তবে এটি কীভাবে এড়িয়ে যায়?**
FLP (Fischer, Lynch, Paterson, 1985) প্রমাণ করে যে message delay-এর কোনো সীমা ছাড়া একটি সম্পূর্ণ asynchronous network-এ, একটিমাত্র node ব্যর্থ হতে পারলেও, কোনো deterministic algorithm safety এবং termination উভয়টিই একসাথে নিশ্চিত করতে পারে না—আপনি সবসময় একটি ধীরগতির node আর একটি মৃত node-এর মধ্যে পার্থক্য করতে পারবেন না। Paxos এবং Raft এটি খণ্ডন করে না; তারা বাস্তবে partial synchrony ধরে নিয়ে এটি এড়িয়ে যায় (message সাধারণত একটি যুক্তিসঙ্গত সময়ের মধ্যে পৌঁছায়) এবং retry বা নতুন election trigger করতে timeout ব্যবহার করে, এই ধরে নিয়ে যে একটি প্যাথলজিক্যাল network-এ তারা তাত্ত্বিকভাবে আটকে যেতে পারে, যদিও বাস্তব deployment-এ এটি প্রায় কখনো ঘটে না।

**৩. একটি 7-node Raft cluster-এ, একটি log entry commit হওয়ার আগে কতগুলো node-কে তা স্বীকার (acknowledge) করতে হবে, এবং কেন?**
৪টি node (Leader নিজেকে সহ)—একটি majority, যা গণনা করা হয় `floor(7/2) + 1 = 4` হিসেবে। এটি প্রয়োজন কারণ 7টি node-এর মধ্যে যেকোনো দুটি majority set-কে অন্তত একটি node-এ ওভারল্যাপ করতেই হবে; সেই নিশ্চিত ওভারল্যাপই নিশ্চিত করে যে ভবিষ্যতের একজন Leader (একটি ভিন্ন majority দিয়ে নির্বাচিত) সবসময় পূর্বে committed entry দেখবে এবং তা নীরবে বাদ দিতে পারবে না।

**৪. একটি network partition যখন একটি 5-node Raft cluster-কে একটি 3-node group এবং একটি 2-node group-এ বিভক্ত করে দেয়, তখন কী ঘটে?**
3-node পক্ষের কাছে এখনও একটি majority (5-এর মধ্যে 3) আছে, তাই এটি স্বাভাবিকভাবে একজন Leader নির্বাচন/বজায় রাখতে পারে এবং নতুন log entry commit করতে পারে। 2-node পক্ষ একটি majority গঠন করতে পারে না, তাই সেখানে একজন Candidate হওয়া যেকোনো node একটি election না জিতেই বারবার timeout হতে থাকবে, এবং সেই পক্ষে কোনো write commit করা যাবে না। এটি ইচ্ছাকৃত design: এটি consistency বজায় রাখতে এবং split-brain প্রতিরোধ করতে সংখ্যালঘু পক্ষের availability বিসর্জন দেয়।

**৫. প্রমাণযোগ্যভাবে সঠিক হওয়া সত্ত্বেও, Paxos-কে কেন সঠিকভাবে implement করা কুখ্যাতভাবে কঠিন বলে মনে করা হয়?**
Base Paxos protocol শুধু একটি একক value নিয়ে একমত হওয়ার পদ্ধতি সংজ্ঞায়িত করে এবং ইচ্ছাকৃতভাবে বাস্তব বিষয়গুলো বাদ দেয়—কীভাবে দক্ষতার সাথে সিদ্ধান্তের একটি ধারাবাহিক sequence চালানো যায়, কীভাবে দ্বন্দ্বরত proposer এড়াতে একটি স্থিতিশীল proposer বেছে নেওয়া যায়, কীভাবে log compaction এবং membership পরিবর্তন সামলানো যায়। Implementer-দের এই সবকিছুর সমাধান নিজেদেরই উদ্ভাবন করতে হয় (প্রায়ই "Multi-Paxos" variant হিসেবে), এবং সেই extension গুলোতে সূক্ষ্ম ভুল করা সহজ এবং ধরা কঠিন, যা ঠিক সেই ফাঁক যা Raft এই mechanism গুলো স্পষ্টভাবে সংজ্ঞায়িত করে বন্ধ করার জন্য ডিজাইন করা হয়েছিল।

**৬. একজন Raft Follower-এর election timeout শেষ হয়ে গেলে ধাপে ধাপে কী ঘটে তা বর্ণনা করুন।**
- এটি Candidate state-এ পরিবর্তিত হয় এবং বর্তমান term সংখ্যা বাড়ায়।
- এটি নিজেকে vote দেয় এবং একটি নতুন random timeout দিয়ে নিজের election timer রিসেট করে।
- এটি তার সর্বশেষ log index/term সহ সব অন্য node-এর কাছে RequestVote RPC পাঠায়।
- যদি এটি cluster-এর একটি majority-র কাছ থেকে vote পায়, তাহলে এটি সেই term-এর জন্য Leader হয়ে যায় এবং heartbeat পাঠাতে শুরু করে।
- যদি এটি জানতে পারে যে অন্য কোনো node ইতিমধ্যে সমান বা বেশি term-এর জন্য Leader, অথবা যদি vote ভাগ হয়ে যায় এবং কেউ না জেতে, তাহলে এটি Follower/Candidate-এ ফিরে যায় (বা থেকে যায়) এবং প্রক্রিয়াটি একটি নতুন randomized timeout দিয়ে আবার চেষ্টা করে।

**৭. প্রতিটি node-এর জন্য একটি fixed timeout-এর বদলে Raft কেন randomized election timeout ব্যবহার করে?**
যদি প্রতিটি Follower একই fixed timeout ব্যবহার করত, তাহলে একজন Leader ব্যর্থ হওয়ার ঠিক পরপরই অনেক node সম্ভবত একই মুহূর্তে Candidate হয়ে যেত, বারবার vote ভাগ হয়ে যেত এবং election-এর convergence অনির্দিষ্টকালের জন্য বিলম্বিত হতো। একটি পরিসরের মধ্যে timeout-কে randomize করা পরিসংখ্যানগতভাবে সম্ভাব্য করে তোলে যে একটি node প্রথমে timeout হবে, Candidate হয়ে যাবে, এবং অন্যরা নিজেদের election শুরু করার আগেই একটি majority সংগ্রহ করে ফেলবে, তাই প্রায় সব ক্ষেত্রে election দ্রুত সমাধান হয়।

**৮. Paxos-এ, একজন Proposer, Promise response পাওয়ার পর, কেন তার নিজের পছন্দের value-এর বদলে যেকোনো Acceptor দ্বারা ইতিমধ্যে accept করা সর্বোচ্চ-সংখ্যাযুক্ত proposal-এর value প্রস্তাব করতে বাধ্য?**
যদি একটি Acceptor-এর Promise response ইঙ্গিত দেয় যে এটি ইতিমধ্যে একটি আগের proposal number-এর অধীনে কোনো value V accept করেছে, তাহলে সেই value V হয়তো ইতিমধ্যে একটি majority দ্বারা chosen হওয়ার পথে আছে (অথবা হয়তো ইতিমধ্যে chosen হয়ে গেছে, যা Proposer কেবল জানে না)। V ছাড়া অন্য কিছু প্রস্তাব করলে একই instance-এর জন্য দুটি ভিন্ন value chosen হওয়ার ঝুঁকি থাকে, যা consensus safety লঙ্ঘন করবে—তাই Proposer সর্বোচ্চ-সংখ্যাযুক্ত পূর্বে accepted value "গ্রহণ" করতে বাধ্য, যাতে এই invariant বজায় থাকে যে সর্বোচ্চ একটি value কখনো chosen হতে পারে।

**৯. Raft consensus-কে স্পষ্টভাবে কোন তিনটি sub-problem-এ বিভক্ত করে, এবং সেই বিভাজন কেন Raft-কে Paxos-এর চেয়ে সহজে যুক্তি দেওয়ার উপযোগী করে তোলে?**
Leader election, log replication, এবং safety। প্রতিটি বিষয়কে তার নিজস্ব স্পষ্টভাবে সংজ্ঞায়িত mechanism বরাদ্দ করে (randomized-timeout election; Leader-চালিত AppendEntries replication; safety-র জন্য একটি স্পষ্ট election restriction এবং commit rule), Raft "generic" Paxos-এর অস্পষ্টতা এড়ায়, যেখানে এই সব বিষয় বিমূর্ত Proposer/Acceptor মডেলে জড়িয়ে থাকে এবং implementer-দের নিজেরাই সমাধান করার জন্য রেখে দেওয়া হয়।

**১০. Paxos (বা একটি Paxos-এর মতো protocol) ব্যবহার করে এমন একটি বাস্তব-জগতের system এবং Raft ব্যবহার করে এমন একটি system-এর নাম বলুন। একটি team আজ একটির বদলে অন্যটি কেন বেছে নিতে পারে?**
Google Spanner এবং Chubby Paxos-ভিত্তিক replication ব্যবহার করে, এবং ZooKeeper ব্যবহার করে ZAB, একটি Paxos-এর মতো protocol; etcd, Consul, এবং CockroachDB Raft ব্যবহার করে। আজ একটি নতুন system তৈরি করা একটি team সাধারণত Raft-এর দিকে ঝুঁকবে কারণ এর স্পষ্ট leader election এবং log replication নিয়মগুলো সঠিকভাবে implement, test, এবং debug করা অনেক সহজ—Paxos মূলত সেই team গুলো বেছে নেয় যারা একটি বিদ্যমান Paxos-ভিত্তিক system উত্তরাধিকারসূত্রে পেয়েছে বা একটি ইতিমধ্যে পরিপক্ব implementation optimize করছে।

**১১. Split-brain কী, এবং নির্দিষ্টভাবে majority-quorum প্রয়োজনীয়তা কীভাবে এটি প্রতিরোধ করে?**
Split-brain হলো যখন একটি network partition একটি cluster-এর দুই বা ততোধিক উপসেটকে প্রত্যেকে একমাত্র কর্তৃত্ব হিসেবে কাজ করতে বাধ্য করে, স্বাধীনভাবে write গ্রহণ করে এবং system-এর state-কে বিচ্যুত করে। যেহেতু একটি majority quorum-এর জন্য একটি fixed-size cluster-এর অর্ধেকের বেশি প্রয়োজন, এবং একই cluster-এর যেকোনো দুটি majority উপসেটকে অন্তত একটি node শেয়ার করতেই হবে, তাই দুটি বিচ্ছিন্ন group-এর একই সময়ে উভয়েরই majority দাবি করা গাণিতিকভাবে অসম্ভব—তাই যেকোনো partition-এর একটি সময়ে সর্বোচ্চ একটি পক্ষই progress করতে পারে।

**১২. Raft-এ, একটি পুরনো (out-of-date) log সহ একটি node কি কখনো একটি leader election জিততে পারে? এই সমস্যা প্রতিরোধকারী safety mechanism-টি ব্যাখ্যা করুন।**
না—Raft-এর election restriction একজন voter-কে একটি RequestVote request প্রত্যাখ্যান করতে বাধ্য করে যদি candidate-এর log তার নিজের চেয়ে কম up to date হয় (প্রথমে শেষ log entry-র term দিয়ে তুলনা করে, তারপর index দিয়ে)। একজন candidate কেবল একটি majority-র কাছ থেকে vote পেয়ে জিততে পারে, এবং যেহেতু node-গুলোর একটি majority-র কাছে অবশ্যই প্রতিটি পূর্বে committed entry থাকতে হবে, তাই সেই majority-র অন্তত একজন voter-এর কাছে ইতিমধ্যে সবচেয়ে up-to-date log থাকবে—যার মানে সে পিছিয়ে থাকা একজন candidate-কে vote দিতে অস্বীকার করবে, যা নিশ্চিত করে যে একটি পুরনো node Leader হওয়ার জন্য প্রয়োজনীয় vote সংগ্রহ করতে পারবে না।
