# কেন এই বিষয়টি গুরুত্বপূর্ণ: Multi-Region Architecture & Disaster Recovery

> **এক বাক্যে:** প্রতিটি redundancy স্ট্র্যাটেজির একটি failure domain থাকে যা সে টিকে থাকতে পারে না, এবং একটি single-region সিস্টেমের জন্য সেই domain হলো region — যা প্রতি বছর বিশ্বের কোথাও না কোথাও, একাধিকবার ডাউন হয়।

## এই ধারণার আগের জগৎ

আপনার সিস্টেম যথাযথভাবে ইঞ্জিনিয়ার করা: একাধিক instance, একাধিক availability zone, replicated database, automated failover। এটি একটি মেশিন failure, একটি rack failure, এমনকি একটি সম্পূর্ণ zone failure-ও সহ্য করতে পারে।

তারপর একটি পুরো cloud region-এ outage ঘটে — একটি control-plane failure, একটি fiber কাটা, একটি power event, একটি খারাপ configuration push। আপনার zone-গুলো কোনো সাহায্য করে না, কারণ failure domain সেগুলোর উপরে। আপনার backup একই region-এ। আপনার DNS একটি load balancer-এর দিকে নির্দেশ করছে যা আর নেই। আপনি ততক্ষণ ডাউন থাকবেন যতক্ষণ provider-এর সময় লাগে, যা আপনি নিয়ন্ত্রণ করেন না এবং অনুমানও করতে পারেন না, এবং অপেক্ষা করা ছাড়া আপনার কাছে কোনো পদক্ষেপ নেই।

এই গল্পের দ্বিতীয় সংস্করণটি আরও খারাপ কারণ এটি নিজেরই তৈরি করা: কেউ `WHERE` clause ছাড়া একটি `DELETE` চালায়, বা একটি খারাপ migration একটি table-কে corrupt করে দেয়, এবং এটি সাথে সাথেই প্রতিটি zone-এর প্রতিটি replica-য় replicate হয়ে যায়। Replication বিশ্বস্তভাবে ভুলটিকেই কপি করে। একটি logical error-এর বিরুদ্ধে redundancy কোনো সুরক্ষাই দেয় না — শুধু backup দেয়, এবং তাও শুধু যদি কেউ সেগুলো restore করার পরীক্ষা করে থাকে।

## এটি যে সমস্যাগুলোর সমাধান করে

### ১. আপনার architecture-এর চেয়ে বড় একটি failure domain
**আপনি যা দেখেন:** কোনো recovery path ছাড়াই সম্পূর্ণ outage, সম্পূর্ণভাবে একটি তৃতীয় পক্ষের উপর নির্ভরশীল।

**কেন এটি ঘটে:** প্রতিটি component একটি region-এর ভেতরে থাকে, তাই region নিজেই একটি single point of failure।

**Multi-region যেভাবে এটি সমাধান করে:** infrastructure এবং data অন্তত দুটি ভৌগোলিকভাবে পৃথক region-এ বিদ্যমান থাকে, এবং একটি failed region থেকে traffic সরিয়ে নেওয়ার একটি mechanism থাকে। এখন failure domain যেকোনো একক provider region-এর চেয়ে বড়।

### ২. কারও দ্বারা পরিমাপ না করা recovery প্রত্যাশা
**আপনি যা দেখেন:** "আমাদের backup আছে" — একটি সম্পূর্ণ disaster recovery plan হিসেবে, restore করতে কতক্ষণ লাগবে বা কতটা data হারানো যাবে তার কোনো উত্তর ছাড়াই।

**কেন এটি ঘটে:** recovery গুণগতভাবে (qualitatively) আলোচনা করা হয়, তাই plan এবং ব্যবসার সহনশীলতার মধ্যে ফাঁকটি অদৃশ্য থাকে যতক্ষণ না সেটি গুরুত্বপূর্ণ হয়ে ওঠে।

**RPO এবং RTO যেভাবে এটি সমাধান করে:** **Recovery Point Objective** হলো আপনি কতটা data হারাতে পারেন (nightly backup মানে 24 ঘণ্টা পর্যন্ত)। **Recovery Time Objective** হলো আপনি কতক্ষণ ডাউন থাকতে পারেন। এই দুটি সংখ্যা architecture এবং তার খরচ নির্ধারণ করে, এবং সেগুলোতে প্রকৃত সংখ্যা বসানোই পুরো planning exercise। চার ঘণ্টার RTO এবং এক ঘণ্টার RPO মানে একটি warm standby; পাঁচ মিনিটের RTO এবং প্রায়-শূন্য RPO মানে continuous replication সহ active-active, এবং একটি সম্পূর্ণ ভিন্ন budget।

### ৩. একটি global user base-এর জন্য latency
**আপনি যা দেখেন:** বিশ্বের অপর প্রান্তের ব্যবহারকারীরা প্রতিটি request-এ কয়েকশ মিলিসেকেন্ড অতিরিক্ত latency অনুভব করেন।

**কেন এটি ঘটে:** পদার্থবিজ্ঞান। দুটি মহাদেশের মধ্যে একটি round trip-এর একটি সীমা আছে যা আপনি অতিক্রম করে optimize করতে পারবেন না।

**Multi-region যেভাবে এটি সমাধান করে:** নিকটতম region থেকে ব্যবহারকারীদের পরিবেশন করলে 300 ms-এর round trip 20 ms-এ পরিণত হয়। read-heavy workload-এর জন্য, regional read replica এবং একটি CDN মিলে multi-region write-এর পুরো জটিলতা ছাড়াই এই সুবিধার বেশিরভাগ অংশ দেয়।

### ৪. Data residency এবং compliance প্রয়োজনীয়তা
**আপনি যা দেখেন:** একটি আইনি প্রয়োজনীয়তা যে EU ব্যবহারকারীদের ব্যক্তিগত data EU-তে সংরক্ষিত এবং প্রক্রিয়াজাত হতে হবে, আর একটি architecture যেখানে একটি মাত্র database Virginia-তে।

**কেন এটি ঘটে:** নিয়ন্ত্রণ (regulation) design-এর একটি input ছিল না।

**Multi-region যেভাবে এটি সমাধান করে:** ব্যবহারকারীর জুরিসডিকশন অনুযায়ী routing সহ region-scoped data। এটি প্রায়শই একটি optimization নয়, বরং একটি কঠোর প্রয়োজনীয়তা, এবং এটি পরে যোগ করা ব্যয়বহুল।

## যে মূল্য আপনাকে দিতে হয়

এই কোর্সে multi-region সবচেয়ে ব্যয়বহুল architectural সিদ্ধান্ত, এবং সৎ হিসাব-নিকাশটি গুরুত্বপূর্ণ:

- **খরচ কমপক্ষে প্রায় দ্বিগুণ হয়** — এবং cross-region replication সহ active-active এর উপর যথেষ্ট পরিমাণে data transfer charge যোগ করে।
- **Write-এই এটি সত্যিকার অর্থে কঠিন হয়ে ওঠে।** Multi-region *read* তুলনামূলকভাবে সহজ। Multi-region *write* একটি প্রকৃত পছন্দে বাধ্য করে: একটি একক write region (সরল, সঠিক, কিন্তু আপনার অর্ধেক ব্যবহারকারীর জন্য অনেক দূরে এবং failover প্রয়োজন), অথবা একাধিক region-এ গৃহীত write (সর্বত্র দ্রুত, এবং এখন আপনাকেই conflict resolution সামলাতে হবে, যা intercontinental latency-তে multi-primary সমস্যা)।
- **CAP অনিবার্য এবং ব্যয়বহুল হয়ে ওঠে।** Cross-region latency 50-150 ms, তাই synchronous replication প্রতিটি write-কে ধীর করে দেয়, এবং asynchronous replication মানে একটি region failure সাম্প্রতিক write হারায়। এমন কোনো configuration নেই যা এই পছন্দ এড়িয়ে যায়।
- **Operational জটিলতা বহুগুণ বেড়ে যায়।** Deploy, migration, feature flag, এবং configuration region জুড়ে সমন্বিত করতে হয়, এবং region-গুলো drift করতে পারে।
- **Failover-ই ঝুঁকিপূর্ণ অংশ।** Automatic failover-এ region জুড়ে flapping এবং split-brain-এর ঝুঁকি থাকে; manual failover ধীর এবং চাপের মধ্যে মানুষের উচ্চ-ঝুঁকির সিদ্ধান্তের উপর নির্ভরশীল। যেভাবেই হোক, **যে failover কখনো পরীক্ষা করা হয়নি সেটি কাজ করে না** — একটি প্রকৃত disaster এলে এটাই সবচেয়ে সাধারণ আবিষ্কার।
- **Backup অপরিহার্য এবং পৃথক থেকে যায়।** Multi-region infrastructure failure-এর বিরুদ্ধে সুরক্ষা দেয়; শুধুমাত্র backup logical corruption-এর বিরুদ্ধে সুরক্ষা দেয়, যা সাথে সাথেই replicate হয়ে যায়। Backup অবশ্যই immutable হতে হবে, আলাদাভাবে সংরক্ষণ করতে হবে, এবং — গুরুত্বপূর্ণভাবে — একটি drill হিসেবে পর্যায়ক্রমে *restore* করতে হবে।

**একটি বাস্তবসম্মত সিঁড়ি, সবচেয়ে সস্তা থেকে সবচেয়ে ব্যয়বহুল:** এক region-এ multi-AZ (বেশিরভাগ প্রকৃত failure কভার করে, কম খরচে) → দ্বিতীয় region-এ backup সহ একটি documented restore (disaster কভার করে, ধীর recovery) → pilot light বা warm standby (দ্রুততর RTO, মাঝারি খরচ) → active-active (সর্বনিম্ন RTO এবং সেরা latency, সর্বোচ্চ খরচ এবং জটিলতা)। বেশিরভাগ organization-এর ইচ্ছাকৃতভাবে একটি ধাপে থামা উচিত, ডিফল্টভাবে উপরে ওঠা নয়।

## কখন এটি প্রয়োজন — এবং কখন প্রয়োজন নয়

| যখন multi-region-এ যাবেন | যখন multi-AZ যথেষ্ট |
|---|---|
| Downtime-এর খরচ প্রতি ঘণ্টায় দ্বিতীয় region-এর খরচের চেয়ে বেশি | আপনি একটি startup যেখানে গতি বেশি গুরুত্বপূর্ণ |
| Contract বা regulation এটি দাবি করে | ব্যবহারকারীরা একটি ভূগোলে কেন্দ্রীভূত |
| আপনার user base সত্যিকার অর্থে global | কয়েক ঘণ্টার recovery time সহনীয় |
| একটি regional outage একটি অস্তিত্বগত ঝুঁকি | আপনি এখনো single-region reliability আয়ত্ত করেননি |

## Interview-এ এটি কেন আসে

Multi-region senior এবং staff-level design round-এ আসে, সাধারণত একটি follow-up হিসেবে: "যদি পুরো region ডাউন হয়ে যায় তাহলে কী হবে?" প্রত্যাশিত উত্তর "আমরা multi-region হব" নয় — এটি একটি যুক্তিসঙ্গত অবস্থান: ব্যবসার প্রয়োজনীয় RPO এবং RTO বলা, মিলে যাওয়া স্ট্র্যাটেজি বেছে নেওয়া (backup-and-restore, warm standby, বা active-active), এবং কঠিন অংশ সম্পর্কে স্পষ্ট হওয়া, যা হলো write এবং conflict resolution। যারা খরচ স্বীকার করে এবং এমন সিস্টেমের জন্য cross-region backup সহ multi-AZ সুপারিশ করে যা আরও বেশি কিছু ন্যায্যতা দেয় না, তারা সাধারণত যারা প্রতিফলিতভাবে active-active-এর দিকে ঝুঁকে পড়ে তাদের চেয়ে ভালো judgment প্রদর্শন করে। failover নিয়মিত পরীক্ষা করা উচিত এই কথা উল্লেখ করা একটি বিবরণ যা interviewer-রা লক্ষ্য করেন।

## এটি কীভাবে সংযুক্ত

এটি সবচেয়ে বড় failure domain-এ **availability and fault tolerance** (topic 5), যা region জুড়ে **replication** (topic 13)-এর উপর নির্মিত এবং **CAP and PACELC** (topic 15)-কে সরাসরি মোকাবিলা করতে বাধ্য করে, কারণ cross-region latency consistency trade-off-কে অনিবার্য করে তোলে। এটি routing-এর জন্য **CDN** (topic 18) এবং geo-aware **load balancing** (topic 7)-এর উপর নির্ভর করে, একাধিক region write গ্রহণ করলে **logical clocks** (topic 41)-এর উপর, এবং failover আসলেই কাজ করে কিনা তা যাচাই করতে **chaos engineering** (topic 46)-এর উপর নির্ভর করে। Data residency **security and compliance** (topic 36)-এর সাথে ফিরে যুক্ত হয়।

**পরবর্তী:** [Design a URL Shortener](../../Module-12-Case-Studies-Interview-Practice/48-design-a-url-shortener/why.md) — প্রকৃত interview সমস্যায় সম্পূর্ণ toolkit প্রয়োগ করা।
