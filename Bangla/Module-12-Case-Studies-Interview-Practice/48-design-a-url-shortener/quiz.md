# অনুশীলন ও ইন্টারভিউ প্রশ্ন

মূল design আলোচনার পরে একজন interviewer যেসব follow-up প্রশ্ন করতে পারেন, তার সংক্ষিপ্ত model উত্তর সহ।

**১. User-দের অনুরোধ করা custom alias আপনি কীভাবে handle করবেন?**
- Custom-alias request-কে auto-generated code থেকে একটি আলাদা write path হিসেবে treat করুন: alias reserve করার আগে অনুরোধ করা alias-টা ইতিমধ্যে ব্যবহৃত কিনা তা database-এ (অথবা দ্রুত negative check-এর জন্য প্রথমে একটি bloom filter-এ) check করুন।
- নেওয়া হয়ে গেলে, একটি conflict error ফেরত দিন এবং user-কে অন্য একটা বেছে নিতে দিন; পাওয়া গেলে, auto-generated code-এর মতো একই schema দিয়ে সেটা লিখুন যাতে read path-এ branch করার দরকার না হয়।
- এটা সেই collision সমস্যা আবার নিয়ে আসে যা counter-based generator এড়ায়, তাই এটা স্বভাবতই একটু ধীর এবং একটি uniqueness constraint বা conditional write দরকার।

**২. হঠাৎ traffic spike-এর মধ্যে hot/viral URL-কে কীভাবে দ্রুত রাখবেন?**
- Cache-aside layer-এর উপর নির্ভর করুন: জনপ্রিয় link LRU eviction-এর অধীনে resident থাকে কারণ সেগুলো ক্রমাগত re-read হয়, তাই spike-এর বেশিরভাগ অংশ database নয়, cache শুষে নেয়।
- একটি চরম spike-এর জন্য (একটি link কয়েক মিনিটে viral হয়ে যাওয়া), সবচেয়ে hot key-গুলোর জন্য network hop সম্পূর্ণভাবে কমাতে shared cache-এর পাশাপাশি প্রতিটি app server-এ একটি ছোট local in-memory cache বিবেচনা করুন।
- Cache node জুড়ে consistent hashing নিশ্চিত করে যে একটি hot key থেকে আসা অতিরিক্ত load, cluster scale হওয়ার সাথে সাথে একটি single shard-কে অসমভাবে overload করে না।

**৩. Redirect ধীর না করে আপনি কীভাবে click analytics যোগ করবেন?**
- Redirect path-এ কখনো synchronously analytics কাজ করবেন না। একটি হালকা click event (short code, timestamp, হয়তো referrer/IP) একটি message queue-তে log করুন।
- একটি আলাদা downstream consumer/pipeline এই event-গুলোকে analytics query-র জন্য optimize করা একটি data store-এ aggregate করে।
- এটা redirect-এর critical path-কে একটি cache lookup এবং একটি HTTP 301/302-এই সীমাবদ্ধ রাখে, analytics-কে eventually consistent হিসেবে গ্রহণ করা হয়।

**৪. Short code শেষ হয়ে গেলে (key exhaustion) কী হয়?**
- Base62 এবং 7 character দিয়ে, key space প্রায় 3.5 trillion code, যা 5 বছরে প্রয়োজনীয় 30 বিলিয়নের চেয়ে আরামসে বেশি, তাই এই scale-এ exhaustion বাস্তবসম্মত নয়।
- যদি এটা কখনো সত্যিকারের উদ্বেগের বিষয় হয়ে ওঠে, তাহলে সমাধান হলো encoding scheme redesign না করে code-এর length বাড়ানো (একটি character যোগ করলে space 62 গুণ বেড়ে যায়)।
- Expired/deleted link থেকে code পুনরুদ্ধার করা সম্ভব কিন্তু জটিলতা যোগ করে (নিশ্চিত করতে হবে code কোথাও cache বা reference করা নেই) এবং base62 ইতিমধ্যে যে headroom দেয় তা বিবেচনা করলে খুব কমই এর মূল্য আছে।

**৫. SQL নাকি NoSQL — আপনি আসলে কোনটা বেছে নেবেন, এবং কেন?**
- মূল access pattern একটি সহজ key lookup (short code -> long URL), কোনো join বা জটিল transaction ছাড়াই, যা সহজ horizontal scaling-এর জন্য DynamoDB বা Cassandra-র মতো একটি NoSQL key-value store-কে অনুকূল করে।
- একটি sharded relational database (MySQL/PostgreSQL)-ও defensible, বিশেষত যদি custom-alias uniqueness-এর জন্য একটি strong constraint দরকার হয়, বা team-এর ইতিমধ্যেই relational operational expertise থাকে।
- শেষ পর্যন্ত এটা একটি CAP theorem trade-off: এখানে NoSQL option-গুলো সাধারণত eventual consistency সহ availability এবং partition tolerance (AP)-কে অগ্রাধিকার দেয়, যা click count-এর জন্য ঠিক আছে কিন্তু write time-এ alias uniqueness-এর জন্য অতিরিক্ত সতর্কতা দরকার।

**৬. Malicious বা duplicate URL submission থেকে আপনি system-কে কীভাবে রক্ষা করবেন?**
- জমা দেওয়া URL validate এবং sanitize করুন (সঠিকভাবে গঠিত, SSRF-স্টাইল abuse প্রতিরোধ করতে internal/private IP range-এর দিকে নির্দেশ করছে না)।
- Shorten করার আগে submission-গুলোকে একটি blocklist বা threat-intelligence/malware-URL service-এর বিপরীতে check করুন, এবং পর্যায়ক্রমে পুনরায় check করার কথা বিবেচনা করুন যেহেতু একটি URL পরে malicious হয়ে যেতে পারে।
- Duplicate-এর জন্য, ঐচ্ছিকভাবে লম্বা URL hash করুন এবং দেখুন এটা ইতিমধ্যে shorten করা হয়েছে কিনা, নতুন redundant entry তৈরি না করে বিদ্যমান short code ফেরত দিন — এটা স্বাভাবিকভাবেই storage growth-ও কমায়।

**৭. আপনি কীভাবে expiring link সমর্থন করবেন?**
- প্রতিটি record-এ একটি optional `expiresAt` timestamp store করুন; read path redirect করার আগে সেটা check করে এবং সময় পার হয়ে গেলে একটি "link expired/not found" response ফেরত দেয়।
- Expired entry lazily purge করা যায় (read-এ check করে) অথবা একটি periodic batch job-এর মাধ্যমে যা database থেকে সেগুলো সরায় এবং cache থেকে evict করে।
- Expired link-এর cache entry-র জন্য হয় expiry-র সাথে মেলানো একটি TTL, নয়তো সক্রিয় invalidation দরকার, নাহলে cache তার expiration পার হয়ে যাওয়ার পরও একটি redirect serve করতে পারে।

**৮. আপনি এটা একাধিক region জুড়ে কীভাবে deploy করবেন?**
- প্রতি region-এ app server, cache, এবং database shard deploy করুন, একটি global load balancer বা DNS-based routing (যেমন, latency-based routing) সহ যা user-দের তাদের নিকটতম region-এ পাঠায়।
- Short-code-থেকে-URL mapping region জুড়ে replicate করুন, নতুন তৈরি link-এর জন্য eventual consistency গ্রহণ করে (একটি region-এ তৈরি একটি link অন্য region-এ দেখা যেতে একটু সময় নিতে পারে) সব জায়গায় কম read latency-র বিনিময়ে।
- Consistent hashing প্রতিটি region-এর cache/database cluster-এর মধ্যে সাহায্য করে, যেখানে cross-region replication আলাদাভাবে, সাধারণত asynchronously, handle করা হয়, যাতে এক region-এর write latency অন্য region-এর network round-trip-এর সাথে couple না হয়ে যায়।

**৯. একটি custom alias edit বা delete করা হলে cache invalidation কীভাবে কাজ করে?**
- সক্রিয় invalidation দরকার: update/delete-এ, app server শুধু একটি TTL-এর উপর নির্ভর না করে সংশ্লিষ্ট cache entry স্পষ্টভাবে evict (বা overwrite) করে।
- Cache distributed হলে, invalidation-কে সেই key ধরে রাখা প্রতিটি node/replica-তে পৌঁছাতে হবে, শুধু app server যেটার সাথে কথা বলে সেটাতে নয় — cache node জুড়ে একটি pub/sub invalidation message একটি সাধারণ pattern।
- Invalidation সম্পূর্ণ হওয়ার আগে, একটি সংক্ষিপ্ত window থাকে যেখানে একটি stale redirect serve হতে পারে; product requirement edit-এ তাৎক্ষণিক consistency দাবি না করলে এটা একটি গ্রহণযোগ্য trade-off।

**১০. শুধু URL hash করার বদলে একটি counter-based key generation service কেন বেছে নেবেন?**
- Hash-based generation (যেমন, একটি MD5/SHA digest base62-encode করা)-এর জন্য প্রতিটি write-এ একটি collision check-and-retry loop দরকার, এবং table পূর্ণ হতে থাকলে retry আরও ঘন ঘন হয়।
- একটি counter-based পদ্ধতি নির্মাণগতভাবেই uniqueness guarantee করে, তাই write path-এ কোনো retry loop-ই নেই।
- Trade-off টা operational: একটি counter service একটি ছোট stateful component যেটা নিজেকেই scale করতে হয়, সাধারণত counter-টাকেই shard করে (যেমন, odd/even range, অথবা Snowflake-style একটি region/machine ID embed করে) অথবা প্রতিটি app server-কে pre-allocated ID batch দিয়ে।

**১১. Abuse প্রতিরোধ করতে shorten endpoint-কে আপনি কীভাবে rate-limit করবেন?**
- Request app server-এ পৌঁছানোর আগে, load balancer/API gateway layer-এ প্রতি API key, user account, বা source IP-তে একটি token bucket বা sliding-window rate limiter প্রয়োগ করুন।
- একটি client তার quota অতিক্রম করলে একটি `Retry-After` header সহ 429 Too Many Requests ফেরত দিন।
- এটা scripted abuse থেকে key-space exhaustion-এর বিরুদ্ধে এবং harmful destination-এ mass redirect তৈরি করতে service ব্যবহার করা কোনো malicious actor-এর বিরুদ্ধে — দুটো থেকেই রক্ষা করে।

**১২. প্রাথমিক প্রক্ষেপণের চেয়ে data অনেক বেশি বাড়লে আপনি database কীভাবে scale করবেন?**
- Short code-এর উপর consistent hashing ব্যবহার করে database shard করুন, যাতে নতুন shard যোগ করলে সম্পূর্ণ rebalance trigger না হয়ে শুধু key-এর একটা ছোট অংশ remap হয়।
- ভালো key distribution আছে এমন একটি hash function বেছে নিয়ে shard-গুলোকে মোটামুটি সুষম রাখুন, এবং hash ring-এ virtual node ব্যবহার করুন যাতে অসম physical shard capacity hot spot তৈরি না করে।
- Sharding-কে cache-aside layer-এর সাথে combine করুন যাতে বেশিরভাগ read traffic কখনোই database-এ না পৌঁছায়, প্রতিটি shard-এর প্রকৃত query load তার theoretical capacity-র অনেক নিচে রাখা যায়।
