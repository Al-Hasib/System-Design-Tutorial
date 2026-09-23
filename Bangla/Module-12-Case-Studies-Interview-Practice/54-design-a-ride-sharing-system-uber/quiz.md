# অনুসরণমূলক Interview প্রশ্ন (Follow-Up Interview Questions)

**১. আপনি কীভাবে surge pricing পরিচালনা করবেন?**
প্রতিটি geohash cell-এ open ride request এবং available driver-এর অনুপাত ক্রমাগত track করুন। যখন demand একটি threshold-এর বাইরে supply-কে ছাড়িয়ে যায়, Payment Service সেই cell-এ fare-এ একটি surge multiplier প্রয়োগ করে, যা একইসাথে সীমিত driver capacity রেশনিং করে (কিছু rider অপেক্ষা করেন বা বেশি দেন) এবং কাছাকাছি driver-দের hot zone-এ পুনঃস্থাপিত হতে উৎসাহিত করে। Surge ঘন ঘন পুনর্গণনা করা উচিত (যেমন প্রতি ১-২ মিনিটে) এবং দ্রুত price ওঠানামা এড়াতে smooth করা উচিত।

**২. ঘন এলাকায় আপনি কীভাবে rider এবং driver-দের দক্ষতার সাথে match করবেন?**
Geospatial index (geohash বা quadtree) ব্যবহার করে প্রথমে candidate search-কে একটি ছোট radius-এ সীমাবদ্ধ করুন, driver না পেলেই কেবল সম্প্রসারণ করুন। ঘন এলাকায়, candidate list-এর আকার সীমিত রাখুন এবং একটি ব্যয়বহুল optimization-এর বদলে একটি সস্তা heuristic (ETA, distance) দিয়ে rank করুন, কারণ theoretically সেরা driver খুঁজে বের করার চেয়ে latency বেশি গুরুত্বপূর্ণ।

**৩. আপনি কীভাবে একজন driver-এর double-booking প্রতিরোধ করবেন?**
"Reserve driver"-কে একটি optimistic assignment না করে একটি একক atomic, conditional অপারেশন করুন (যেমন driver-status field-এ একটি compare-and-set, বা trip store-এ একটি conditional write)। এটি trip Saga-এর প্রথম ধাপ: শুধুমাত্র একটি সমসাময়িক request একজন driver-কে "available" থেকে "reserved"-এ পরিবর্তন করতে পারে, এবং হেরে যাওয়া request-গুলো সাথে সাথেই পরবর্তী candidate-এর বিপরীতে পুনরায় match হয়।

**৪. Trip-এর মাঝপথে বা trip সম্পন্ন হওয়ার সময় payment ব্যর্থতা আপনি কীভাবে পরিচালনা করবেন?**
Payment charge-কে একটি নির্দিষ্ট compensation path সহ একটি saga step হিসেবে বিবেচনা করুন: ব্যর্থ হলে, সংরক্ষিত একটি backup payment method দিয়ে retry করুন; সেটাও ব্যর্থ হলে, ইতিমধ্যে শারীরিকভাবে ঘটে যাওয়া একটি trip-কে "undo" করার চেষ্টা না করে trip-টিকে completed কিন্তু payment-pending হিসেবে চিহ্নিত করুন এবং offline collection/dunning-এর জন্য flag করুন। Charge call সবসময় trip ID-এর সাথে সংযুক্ত একটি idempotency key বহন করে যাতে retry কখনো duplicate charge-এর ঝুঁকি না নেয়।

**৫. আপনি কীভাবে একজন rider-এর double-charging প্রতিরোধ করবেন?**
Payment Service-এ পাঠানো প্রতিটি charge request-এ একটি idempotency key (trip ID থেকে নির্ধারণমূলকভাবে উদ্ভূত) সংযুক্ত করুন। যদি একই request দুইবার পাওয়া যায় — যেমন একটি network timeout এবং client retry-এর কারণে — Payment Service duplicate key-টি চিনতে পারে এবং দ্বিতীয়বার চার্জ প্রক্রিয়া করার বদলে মূল charge ফলাফল ফেরত দেয়।

**৬. GPS signal loss-এর জন্য (যেমন টানেল, ঘন শহুরে canyon) আপনি কীভাবে ডিজাইন করবেন?**
Driver app-কে স্থানীয়ভাবে location ping buffer করতে এবং শেষ পরিচিত trajectory interpolate (dead reckoning) করতে দিন যাতে ম্যাপে driver জমে যাওয়া বা teleport হওয়ার মতো না দেখায়। পুনঃসংযোগে, buffered ping flush করুন যাতে trip-এর ঐতিহাসিক trace সঠিক থাকে। Trip Service-কে সাথে সাথে ব্যর্থ না করে একটি location-update timeout (যেমন 30-60 সেকেন্ড) সহ্য করা উচিত, তারপর trip-কে "possibly stalled" হিসেবে flag করা উচিত।

**৭. আপনি কীভাবে বিশ্বব্যাপী location update স্কেল করবেন?**
Consistent-hashing key হিসেবে geohash prefix ব্যবহার করে location store-কে ভৌগোলিকভাবে shard করুন, যাতে প্রতিটি অঞ্চলের driver traffic কাছাকাছি, অঞ্চল-স্থানীয় node-এ পড়ে। যে driver-দের সেবা দেয় তাদের কাছাকাছি regional Location Service cluster চালান (latency ও cross-region bandwidth কমিয়ে), এবং location data-কে availability-favored (AP) হিসেবে বিবেচনা করুন যাতে একটি আঞ্চলিক partition অন্যত্র matching বন্ধ না করে দেয়।

**৮. Trip flow-এর জন্য two-phase commit (2PC)-এর বদলে একটি Saga কেন ব্যবহার করবেন?**
2PC-এর জন্য transaction-এর পুরো সময়কাল জুড়ে সব অংশগ্রহণকারী service-এ lock ধরে রাখতে হয় — যখন একটি "transaction" একটি ২০ মিনিটের শারীরিক গাড়ির যাত্রা জুড়ে বিস্তৃত থাকে তখন এটি অব্যবহারিক। একটি Saga flow-কে local transaction-এ ভাঙে (driver সংরক্ষণ, match নিশ্চিতকরণ, trip পরিচালনা, payment charge), প্রতিটি আলাদাভাবে commit করা হয়, যেকোনো ধাপে ব্যর্থতার জন্য নির্ধারিত compensating action সহ — কঠোর atomicity-কে availability এবং ব্যবহারিক failure handling-এর জন্য বিনিময় করে।

**৯. Driver location data-এর জন্য আপনি কোন consistency model ব্যবহার করবেন, এবং কেন?**
Eventual consistency / AP। একজন driver-এর position এক বা দুই সেকেন্ড পুরনো হওয়া ব্যবহারকারীর কাছে অনুভূত হয় না এবং কোনো correctness সমস্যা তৈরি করে না, তাই আমরা সেই data-তে কঠোর consistency guarantee-এর চেয়ে উচ্চ write throughput ও partition-এর অধীনে availability-কে অগ্রাধিকার দিই (500K+ update/sec)।

**১০. Trip state এবং payment data-এর জন্য আপনি কোন consistency model ব্যবহার করবেন, এবং কেন?**
Strong consistency / CP। "এই driver কি সংরক্ষিত" বা "এই trip-টি কি চার্জ করা হয়েছে"-এর একটি অসামঞ্জস্যপূর্ণ দৃষ্টিভঙ্গি সরাসরি double-booking বা double-charging-এর দিকে নিয়ে যায় — প্রকৃত আর্থিক ও বিশ্বাসের খরচ — তাই আমরা এই সংকীর্ণ, কম-ভলিউমের data-slice-এ correctness-এর বিনিময়ে কমে যাওয়া availability মেনে নিই (যেমন একটি partition-এর সময় একটি write প্রত্যাখ্যান করা)।

**১১. Raw distance-এর বাইরে একাধিক candidate driver-কে একটি match-এর জন্য আপনি কীভাবে rank করবেন?**
একাধিক signal-কে একটি score-এ একত্রিত করুন: pickup-এ আনুমানিক সময় (সরল-রেখা দূরত্ব নয়, road network এবং traffic হিসাব করে), driver rating, driver-এর বর্তমান trip-completion streak/fatigue নিয়ম, এবং rider-এর পছন্দের সাথে vehicle type match। Scoring function দ্রুত রাখুন (sub-100ms) যেহেতু এটি matching latency budget-এর মধ্যে চলে।

**১২. একজন driver match-এর মাঝপথে offline হয়ে গেলে বা connectivity হারালে আপনি কীভাবে পরিচালনা করবেন?**
Driver location ping-এ একটি heartbeat/timeout ব্যবহার করুন; যদি একজন reserved driver পরপর কয়েকটি heartbeat মিস করেন, Trip Service reservation-কে expired হিসেবে বিবেচনা করে, driver-কে ছেড়ে দেয়, এবং rider-কে পরবর্তী candidate-এর সাথে পুনরায় match করতে Saga-এর compensation path ট্রিগার করে — request-টিকে নীরবে আটকে না রেখে rider-কে বিলম্বের কথা জানিয়ে।
