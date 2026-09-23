# অনুশীলনী ও ইন্টারভিউ প্রশ্ন (Practice & Interview Questions)

**১. Rate limiting কোন সমস্যা সমাধান করে, এবং কেন "শুধু আরও সার্ভার যোগ করা" একটি পর্যাপ্ত উত্তর নয়?**
Rate limiting fairness রক্ষা করে (একজন client অন্যদের ক্ষুধার্ত রাখতে পারে না), availability রক্ষা করে (traffic spike, retry storm, এবং অপব্যবহার থেকে রক্ষা করে), এবং cost predictability রক্ষা করে (প্রতি client-এ আপনাকে কতটুকু capacity প্রভিশন করতে হবে তা সীমাবদ্ধ করে)। সার্ভার যোগ করা সাহায্য করে না কারণ একজন একক দুর্ব্যবহারকারী বা দূষিত client আপনি infrastructure স্কেল করতে পারার চেয়ে দ্রুত তার request volume স্কেল করতে পারে, এবং database-এর মতো একটি শেয়ারড resource-এ সীমাহীন traffic শুধু stateless layer স্কেল করে ঠিক করা যায় না।

**২. Fixed window counter-এ boundary সমস্যাটি ব্যাখ্যা করুন। এটা কতটা খারাপ হতে পারে?**
Fixed window নির্দিষ্ট interval boundary-তে (যেমন, প্রতি ৬০ সেকেন্ডে) তার counter রিসেট করে। একটি client একটি window-এর একদম শেষে সম্পূর্ণ limit পাঠাতে পারে এবং পরের window-এর একদম শুরুতে আবার সম্পূর্ণ limit পাঠাতে পারে, যার ফলে boundary জুড়ে একটি ছোট সময়ের মধ্যে উদ্দিষ্ট limit-এর প্রায় ২ গুণ হয়ে যায় — যদিও প্রতিটি পৃথক window-এর count limit-এর মধ্যে ছিল।

**৩. Sliding window log অ্যালগরিদম কীভাবে boundary সমস্যা ঠিক করে, এবং এর জন্য কী মূল্য দিতে হয়?**
এটা প্রতি client-এ প্রতিটি request-এর একটি timestamp সংরক্ষণ করে (যেমন, একটি Redis sorted set-এ), এবং প্রতিটি নতুন request-এ এটা window-এর চেয়ে পুরনো entry বাদ দেয় এবং যা বাকি থাকে তা গণনা করে। এটা কোনো boundary artifact ছাড়া নিখুঁত প্রয়োগ দেয়, কিন্তু memory cost প্রতি client-এ request volume অনুযায়ী রৈখিকভাবে বাড়ে, যা উচ্চ throughput-এ ব্যয়বহুল হয়ে যায়।

**৪. Sliding window counter কী, এবং কেন এটা অনেক সিস্টেমে ব্যবহারিক ডিফল্ট?**
এটা দুটি fixed-window counter রাখে (বর্তমান এবং পূর্ববর্তী) এবং sliding count আনুমানিক করে `current_count + previous_count * overlap_fraction` হিসেবে, যেখানে overlap_fraction হলো পূর্ববর্তী window-এর কতটুকু অংশ এখনও sliding view-এর ভেতরে পড়ে। এটা sliding log-এর নির্ভুলতার কাছাকাছি পৌঁছায় শুধু প্রতি client-এ দুটি counter খরচ করে — কম memory cost-কে যথেষ্ট ভালো accuracy-র সাথে একত্র করে।

**৫. কেন token bucket burst-এর অনুমতি দেয় কিন্তু leaky bucket দেয় না?**
Token bucket *admission-এর হার* নিয়ন্ত্রণ করে: client নিষ্ক্রিয় থাকা অবস্থায় token একটি capacity পর্যন্ত জমা হয়, এবং একটি request admitted হতে শুধু একটি token প্রয়োজন, তাই একটি client তাৎক্ষণিকভাবে একটি burst-এ সমস্ত জমানো token খরচ করতে পারে। Leaky bucket *output-এর হার* নিয়ন্ত্রণ করে: request গুলো queue করে এবং যেভাবেই আসুক না কেন একটি কঠোরভাবে স্থির হারে ব্যাকএন্ডে drain হয়, তাই আগমনের একটি burst-ও fixed leak rate-এ serialized হয়ে বের হয় — এটা কখনো সেই হার ছাড়িয়ে যেতে পারে না।

**৬. ৫০টি API gateway instance জুড়ে ধারাবাহিকভাবে কাজ করে এমন একটি distributed rate limiter ডিজাইন করুন। আপনি কোন পদ্ধতি নেবেন এবং কেন?**
Local প্রতি-instance counter-এর পরিবর্তে একটি একক সত্যের উৎস হিসেবে একটি কেন্দ্রীভূত, atomically-আপডেট হওয়া store (সাধারণত Redis) ব্যবহার করুন। Check-and-increment-কে একটি একক atomic operation হিসেবে প্রয়োগ করুন — হয় fixed-window limiting-এর জন্য Redis `INCR` + `EXPIRE`, অথবা token bucket/sliding window logic-এর জন্য `EVAL`-এর মাধ্যমে execute হওয়া একটি Lua script — যাতে বিভিন্ন gateway instance-এ আঘাত করা concurrent request জুড়ে read-check-write ক্রম race করতে না পারে। এটা নিশ্চিত করে যে global limit ঠিক যেমন সেভাবে প্রয়োগ হয়, একটি request কোন instance-এ পৌঁছায় তা নির্বিশেষে।

**৭. প্রতিটি rate-limited request-এর hot path-এ Redis রাখার ট্রেড-অফ কী?**
এটা প্রতি request-এ একটি network round trip যোগ করে এবং rate limiting-কে Redis-এর availability ও latency-র উপর নির্ভরশীল করে তোলে — যদি Redis ধীর বা down হয়, প্রতিটি গেটেড request প্রভাবিত হয়। বিকল্প হলো approximate local rate limiting (প্রতিটি instance limit-এর একটি ভগ্নাংশ প্রয়োগ করে, বা পর্যায়ক্রমে count sync করে), যা Redis-কে hot path থেকে সরিয়ে দেয় এবং সত্যিকারের global limit-এর উপরে সংক্ষিপ্ত, সীমাবদ্ধ অতিরিক্ত admission অনুমতি দেওয়ার বিনিময়ে latency/availability উন্নত করে।

**৮. একজন client অভিযোগ করে যে তাদের সুশৃঙ্খল অ্যাপ হঠাৎ rate-limited হয়ে যায় যদিও গড়ে প্রতি মিনিটে ১০০-এর কম request পাঠায়। কোন অ্যালগরিদম দোষী হতে পারে, এবং কেন?**
সম্ভবত একটি fixed window counter, এমন একটি client-এর সাথে মিলিত যে প্রতি মিনিটে একই wall-clock boundary-র কাছাকাছি একটি burst-প্রবণ প্যাটার্নে তার request পাঠায়। গড় কম হলেও, নির্দিষ্ট সময় একটি window edge জুড়ে পড়তে পারে এবং limit ট্রিপ করতে পারে। Sliding window counter বা token bucket-এ পরিবর্তন করা এটা মসৃণ করবে কারণ কোনোটিই শুধুমাত্র একটি কঠোর clock-aligned boundary-র ভিত্তিতে traffic-কে শাস্তি দেয় না।

**৯. আপনি কীভাবে Redis-এ token bucket logic atomically প্রয়োগ করবেন? পদ্ধতিটি স্কেচ করুন।**
Bucket-এর শেষ-রিফিল timestamp এবং বর্তমান token count প্রতি client key-তে একটি Redis hash-এর field হিসেবে সংরক্ষণ করুন। একটি Lua script-এ (atomically execute হওয়ার জন্য `EVAL`-এর মাধ্যমে চালানো): শেষ রিফিলের পর থেকে কতটুকু সময় গেছে তা হিসাব করুন, `elapsed * refill_rate` token যোগ করুন যা bucket-এর সর্বোচ্চ capacity-তে সীমাবদ্ধ, তারপর যদি অন্তত একটি token উপলব্ধ থাকে, সেটা decrement করুন এবং request-কে অনুমতি দিন; অন্যথায় প্রত্যাখ্যান করুন। যেহেতু Redis Lua script গুলো single-threaded ভাবে execute করে, এটা ভিন্ন gateway instance থেকে আসা concurrent request-এর মধ্যে race condition এড়ায়।

**১০. কেন AWS API Gateway-এর মতো প্রোডাকশন সিস্টেমগুলো একটি একক limit-এর পরিবর্তে "rate" এবং "burst" উভয় কনফিগারেশন মান উন্মুক্ত করে?**
এটা সরাসরি token bucket প্যারামিটারের সাথে মেলে: "rate" হলো steady-state token রিফিল হার (অনির্দিষ্টকালের জন্য অনুমোদিত টেকসই throughput) এবং "burst" হলো bucket capacity (জমানো token থেকে তাৎক্ষণিকভাবে কতগুলো request admit করা যায়)। উভয়ই উন্মুক্ত করা API consumer-দের বাস্তবসম্মত traffic প্যাটার্ন (যেমন, একটি client যা কাজ batch করে) সামলাতে দেয়, টেকসই rate limit না বাড়িয়ে এবং ব্যাকএন্ড capacity overload-এর ঝুঁকি না নিয়ে।

**১১. কখন আপনি ইচ্ছাকৃতভাবে একটি প্রোডাকশন সিস্টেমের জন্য token bucket-এর চেয়ে leaky bucket বেছে নেবেন?**
যখন downstream সিস্টেম সত্যিকারভাবে burst সহ্য করতে পারে না এবং একটি মসৃণ, স্থির-হার কাজের stream প্রয়োজন — উদাহরণস্বরূপ, একটি fixed-capacity worker queue-কে খাওয়ানো, নিজের কোনো buffering নেই এমন একটি legacy সিস্টেম, বা একটি কঠোর fixed-rate চুক্তি সহ একটি third-party API-তে outbound traffic shape করা। Nginx-এর `limit_req` module একটি বাস্তব উদাহরণ: এটা burst গুলোকে সরাসরি পাস হতে না দিয়ে একটি স্থির outflow rate প্রয়োগ করতে request গুলোকে queue এবং delay করে।

**১২. একটি queue-ভিত্তিক leaky bucket প্রয়োগ ব্যবহার করলে বনাম একটি সরল reject-on-full প্রয়োগ ব্যবহার করলে request semantics-এর কী হয়?**
একটি queue-ভিত্তিক (`burst` সহ) leaky bucket অতিরিক্ত request গুলোকে একটি FIFO queue-তে অপেক্ষা করতে দেয় এবং পরে স্থির leak rate-এ প্রক্রিয়া করে, একটি নিম্ন প্রত্যাখ্যান হার-এর বিনিময়ে যোগ করা latency নেয় — ভালো যখন client রা বিলম্বিত response সহ্য করতে পারে। একটি সরল reject-on-full leaky bucket bucket পূর্ণ হয়ে গেলে তাৎক্ষণিকভাবে একটি error ফেরত দেয় (যেমন, `429`), সম্পূর্ণতার চেয়ে দ্রুত failure এবং predictable resource usage-কে অগ্রাধিকার দেয়, যা latency-সংবেদনশীল সিস্টেমের জন্য উপযুক্ত যেখানে একটি পুরনো, বিলম্বিত response একটি তাৎক্ষণিক প্রত্যাখ্যানের চেয়ে খারাপ।
