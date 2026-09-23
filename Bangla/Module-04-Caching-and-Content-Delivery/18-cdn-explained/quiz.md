# অনুশীলনী ও Interview প্রশ্ন (Practice & Interview Questions)

**১. CDN কোন মূল সমস্যা সমাধান করে, এবং কেন আপনি শুধু আপনার origin data center-এ আরও বেশি server যোগ করে এটি সমাধান করতে পারবেন না?**
একটি CDN user-দের এবং একটি single origin server-এর মধ্যে physical distance-এর কারণে সৃষ্ট latency সমাধান করে — data speed of light দ্বারা সীমাবদ্ধ, তাই দূরবর্তী user-রা সবসময় network round-trip delay অনুভব করেন। একটি origin location-এ আরও বেশি server যোগ করলে সেই physical distance কমে না; আপনার user-দের physically কাছাকাছি server প্রয়োজন, যা ঠিক একটি CDN-এর distributed edge network প্রদান করে।

**২. "origin server," "edge server," এবং "PoP" সংজ্ঞায়িত করুন, এবং এরা কীভাবে সম্পর্কিত তা ব্যাখ্যা করুন।**
Origin server content-এর authoritative, canonical copy ধারণ করে। একটি PoP (Point of Presence) হলো CDN provider দ্বারা পরিচালিত একটি physical data center location। একটি edge server হলো একটি PoP-এর ভেতরের একটি server যা origin content-এর copy cache করে এবং কাছাকাছি user-দের serve করে, শুধুমাত্র একটি cache miss-এ origin-এর সাথে যোগাযোগ করে।

**৩. কেন static content dynamic content-এর তুলনায় edge-এ cache করা সহজ?**
Static content (image, CSS, JS, video) প্রতিটি user-এর জন্য একই, তাই একটি single cached copy একটি region-এর সবাইকে serve করতে পারে। Dynamic content প্রায়ই প্রতি user বা প্রতি request-এ ভিন্ন হয় (personalization, live data), যা একটি single cached copy ব্যাপকভাবে serve করা অনিরাপদ বা ভুল করে তোলে, যদি না অতিরিক্ত কৌশল (micro-caching, প্রতি user segment-এ cache key, edge computing) ব্যবহার করা হয়।

**৪. ব্যবহারকারীদের নিকটতম edge server-এ পরিচালিত করার জন্য DNS-based routing এবং Anycast routing-এর মধ্যে পার্থক্য ব্যাখ্যা করুন।**
DNS-based routing CDN-এর domain-কে DNS query কোথা থেকে এসেছে তার উপর ভিত্তি করে ভিন্ন IP address-এ resolve করে, user-কে একটি কাছাকাছি PoP-এর দিকে পরিচালিত করে। Anycast একই IP address একই সাথে অনেক physical location থেকে announce করে, ইন্টারনেটের নিজস্ব BGP routing-কে প্রতিটি user-এর traffic স্বয়ংক্রিয়ভাবে topologically নিকটতম announcing location-এ পাঠাতে দেয়, DNS resolution behavior-এর উপর নির্ভর না করেই।

**৫. একটি CDN edge-এ একটি cache miss কীভাবে handle করে?**
যখন একটি request করা object নিকটতম edge server-এ cache করা থাকে না, তখন edge server এটি origin server থেকে fetch করে, ভবিষ্যতের request-এর জন্য locally একটি copy cache করে, এবং request করা user-কে content ফেরত দেয়। একই object-এর জন্য পরবর্তী কাছাকাছি request তখন সরাসরি edge cache থেকে serve করা হয়।

**৬. আপনার কোম্পানি এইমাত্র একটি critical bug fix একটি JavaScript file-এ deploy করেছে, কিন্তু user-রা এখনও পুরনো buggy version দেখছেন। CDN-এর মাধ্যমে এটি ঠিক করার দুটি উপায় কী?**
আপনি CDN-এর API বা dashboard-এর মাধ্যমে একটি manual purge/invalidation request দিতে পারেন যাতে সব edge server জুড়ে cached copy অবিলম্বে evict হয়। বিকল্পভাবে, একটি cache-busting strategy ব্যবহার করুন — নতুন file-টি একটি নতুন versioned URL/filename-এর অধীনে deploy করুন (যেমন, একটি content hash সহ) যাতে এটি পুরনো cached one-এর expire হওয়ার উপর নির্ভর না করে একটি একদম নতুন, uncached resource হিসেবে treat করা হয়।

**৭. Micro-caching কী, এবং এটি semi-dynamic content-এর জন্য কেন ব্যবহার করা হয়?**
Micro-caching একটি dynamic response-কে মোটেও cache না করার বদলে edge-এ খুব অল্প সময়ের জন্য (যেমন, কয়েক সেকেন্ড) cache করে। এটি এমন content-এর জন্য উপযোগী যা ঘন ঘন পরিবর্তিত হয় কিন্তু যেখানে অনেক user-কে কয়েক-সেকেন্ড-পুরনো একটি version serve করা গ্রহণযোগ্য, যা CDN-কে অন্যথায় "dynamic" endpoint-এ বিশাল traffic spike শুষে নিতে দেয়।

**৮. Performance ছাড়াও, CDN একটি production system-কে আর কী সুবিধা প্রদান করে?**
CDN origin-এর সামনে একটি buffer হিসেবে কাজ করে, traffic spike শুষে নেয় (যেমন, viral content) এবং malicious traffic filter করে, যা এদের DDoS attack-এর বিরুদ্ধে একটি সাধারণ প্রথম প্রতিরক্ষা রেখা করে তোলে — origin server বেশিরভাগ volume থেকে সুরক্ষিত থাকে সেটা পৌঁছানোর আগেই।

**৯. কোন HTTP mechanism একটি CDN-কে বলে দেয় এটি কতক্ষণ content-এর একটি অংশ cache করতে পারে?**
`Cache-Control` HTTP response header (যেমন, `Cache-Control: max-age=3600`) যা origin দ্বারা set করা হয়, CDN edge server-কে (এবং browser-কে) বলে দেয় origin-এর সাথে revalidate করার আগে তারা কতক্ষণ cached content serve করতে পারে।

**১০. একটি global video streaming platform কেন প্রতিটি region-এ প্রথম request cache trigger করার জন্য অপেক্ষা করার বদলে জনপ্রিয় content edge server-এ আগেই push করতে পারে?**
একটি organic cache miss-এর জন্য অপেক্ষা করার মানে হলো প্রতিটি region-এর প্রথম user-রা পূর্ণ origin latency অনুভব করেন এবং launch-এর মুহূর্তে origin-এ load যোগ করেন, যা একটি high-demand release-এর জন্য (যেমন, একটি নতুন episode) সমস্যাজনক হতে পারে। জনপ্রিয় content আগে থেকে push করা (pre-warming) নিশ্চিত করে যে demand পৌঁছানোর মুহূর্তেই প্রতিটি region-এ ইতিমধ্যে একটি warm cache আছে, একসাথে অনেক cache miss-এর একটি stampede এড়িয়ে।

**১১. Brazil-এর একজন user এবং Japan-এর একজন user উভয়েই আপনার CDN-backed site থেকে একই static image request করেন। তারা কি একই edge server থেকে serve হবেন? কেন বা কেন নয়?**
না — প্রতিটি user তাদের নিজস্ব নিকটতম edge server-এ route করা হয় (DNS-based বা Anycast routing-এর মাধ্যমে), তাই Brazil-এর user-কে Brazil-এর কাছাকাছি একটি PoP থেকে এবং Japan-এর user-কে Japan-এর কাছাকাছি একটি PoP থেকে serve করা হয়, যদিও underlying content এবং origin অভিন্ন। প্রতিটি edge server তার প্রথম cache miss-এর পর স্বাধীনভাবে নিজের copy cache করে।

**১২. একটি CDN আগের video-তে আলোচিত caching strategy-র সাথে (যেমন, cache-aside, TTL-based invalidation) কীভাবে interact করে?**
একটি CDN মূলত HTTP content-এর জন্য একটি specialized, ভৌগোলিকভাবে distributed cache-aside layer: edge server প্রথমে তাদের local cache check করে, এবং একটি miss-এ origin থেকে fetch করে (database query করার সাথে সাদৃশ্যপূর্ণ) এবং cache populate করে। এটি একই মূল invalidation concept ব্যবহার করে — Cache-Control header-এর মাধ্যমে TTL এবং explicit purge — শুধু একটি single application-এর ভেতরের বদলে একটি global, network-edge scale-এ প্রয়োগ করা।
