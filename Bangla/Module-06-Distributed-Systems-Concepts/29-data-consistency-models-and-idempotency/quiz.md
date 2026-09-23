# অনুশীলন ও Interview প্রশ্ন

**১. Strong consistency এবং eventual consistency-এর মধ্যে পার্থক্য কী?**
Strong consistency (linearizability) guarantee করে যে প্রতিটি read সর্বশেষ write দেখে এবং সব client একই real-time order-এ operation observe করে, যেন data-র একটি single copy আছে। Eventual consistency শুধু guarantee করে যে write বন্ধ হলে replica-গুলো একই value-তে converge হবে — convergence-এ কতটা সময় লাগবে বা client কী মধ্যবর্তী order দেখবে তার কোনো bound দেয় না। Strong consistency-এর মূল্য coordination এবং availability; eventual consistency সম্ভাব্য stale read-এর বিনিময়ে availability এবং latency সর্বোচ্চ করে।

**২. একজন user একটি comment post করে এবং সাথে সাথে refresh করে কিন্তু সেটি দেখতে পায় না। কোন consistency guarantee অনুপস্থিত?**
Read-your-writes consistency। User-এর নিজের write সবসময় তার নিজের পরবর্তী read-এ দৃশ্যমান হওয়া উচিত, এমনকি অন্যথায় eventually consistent একটি system-এও। এটি সাধারণত client-এর read-কে তাদের সর্বশেষ write handle করা replica-তে route করে, বা client ইতিমধ্যে দেখেছে এমন একটি version/timestamp track করে ঠিক করা হয়।

**৩. Causal consistency ব্যাখ্যা করুন এবং এটি কেন গুরুত্বপূর্ণ তার একটি উদাহরণ দিন।**
Causal consistency guarantee করে যে যদি operation B, operation A-এর উপর নির্ভরশীল হয় (A "happens-before" B), তাহলে প্রতিটি node B-এর আগে A দেখে; unrelated operation ভিন্ন ভিন্ন replica-তে ভিন্ন ভিন্ন order-এ দেখা যেতে পারে। উদাহরণ: একটি comment reply কখনো সেই comment-এর আগে দেখা উচিত নয় যেটির সে reply, যদিও সেই comment এবং feed-এর অন্য কোথাও একটি unrelated post-কে বিভ্রান্তি ছাড়াই reorder করা যেতে পারে।

**৪. একটি idempotent payment processing API design করুন। আপনি idempotency key হিসেবে কী ব্যবহার করবেন এবং কীভাবে এটি store করবেন?**
Client প্রতিটি logical payment attempt-এর জন্য (প্রতিটি network retry-এর জন্য নয়) একটি unique key (যেমন, একটি UUID) তৈরি করে এবং এটি `Idempotency-Key`-এর মতো একটি header-এ পাঠায়। Server transaction বা conditional write ব্যবহার করে প্রকৃত charge-এর সাথে atomically `{key -> result}` একটি দ্রুত, durable store-এ (TTL-সহ Redis, বা key-এর উপর unique constraint সহ একটি database table) সংরক্ষণ করে। একই key দিয়ে retry হলে, server stored result খুঁজে বের করে এবং payment processor আবার call না করেই সরাসরি সেটি ফেরত দেয়।

**৫. এই module-এ আগে আলোচিত retry pattern-এর জন্য idempotency কেন গুরুত্বপূর্ণ?**
Retry ডিফল্টভাবে অনিরাপদ কারণ একজন client "request কখনো server-এ পৌঁছায়নি" এবং "request সফল হয়েছিল কিন্তু response হারিয়ে গিয়েছিল" এর মধ্যে পার্থক্য করতে পারে না। যদি অন্তর্নিহিত operation idempotent না হয়, একটি retry প্রভাব duplicate করতে পারে (যেমন, double-charging, duplicate order creation)। Idempotency (প্রায়ই idempotency key-এর মাধ্যমে) কোন failure mode আসলে ঘটেছিল তা নির্বিশেষে retry-কে নিরাপদ করে।

**৬. Idempotency-এর দিক থেকে PUT এবং POST-এর মধ্যে পার্থক্য কী, এবং এটি API design-এর জন্য কেন গুরুত্বপূর্ণ?**
PUT-কে HTTP semantics দ্বারা idempotent হিসেবে সংজ্ঞায়িত করা হয় — একই PUT পুনরাবৃত্তি করলে resource একই state-এ থাকা উচিত। POST idempotent নয় — এটি পুনরাবৃত্তি করলে সাধারণত প্রতিবার একটি নতুন resource তৈরি হয়। এটি গুরুত্বপূর্ণ কারণ client এবং infrastructure (proxy, load balancer) transient failure-এ automatically idempotent method retry করতে পারে, কিন্তু idempotency key ছাড়া POST automatically retry করা উচিত নয়, নাহলে side effect duplicate হওয়ার ঝুঁকি থাকে।

**৭. একটি system কেন strong বা eventual consistency-এর বদলে causal consistency বেছে নিতে পারে?**
Causal consistency linearizability-এর সম্পূর্ণ coordination cost না দিয়েই user-রা স্বজ্ঞাগতভাবে প্রত্যাশা করে এমন ordering guarantee ধরে রাখে (যেমন, comment-এর পরে reply, post-এর পরে like)। এটি একটি ব্যবহারিক মধ্যবর্তী সমাধান: strong consistency-এর চেয়ে বেশি available এবং কম latency, একই সাথে pure eventual consistency-এর সবচেয়ে বিভ্রান্তিকর artifact এড়িয়ে যায়, যেখানে causally সম্পর্কিত event out of order দেখা যেতে পারে।

**৮. Causal consistency implement করতে বা concurrent write সনাক্ত করতে vector clock কীভাবে সাহায্য করে?**
একটি vector clock হলো প্রতিটি write-এ সংযুক্ত একটি per-replica counter vector, প্রতিটি local update-এ বৃদ্ধি পায় এবং অন্য replica থেকে update পাওয়ার সময় merge হয়। Vector clock তুলনা করে, একটি system নির্ধারণ করতে পারে যে একটি write অন্যটির happens-before কিনা (সব counter ≤) নাকি সেগুলো concurrent ছিল (কেউই dominate করে না), যা এটিকে হয় causal order সংরক্ষণ করতে দেয় বা resolution-এর জন্য একটি conflict flag করতে দেয় (যেমন, last-write-wins বা application-level merge)।

**৯. DynamoDB কী কী tunable consistency option প্রদান করে, এবং trade-off কী?**
DynamoDB আপনাকে "eventually consistent reads" (default, সস্তা, কম latency, সামান্য stale data ফেরত দিতে পারে — সাধারণত এক সেকেন্ডের মধ্যে) বা "strongly consistent reads" (বেশি latency এবং cost, সব পূর্ববর্তী সফল write প্রতিফলিত করার guarantee, এবং কিছু network issue-এর সময় unavailable) বেছে নিতে দেয়। পছন্দটি প্রতি read request-ভিত্তিতে করা হয়, যা application-কে প্রতি use case-এর জন্য সঠিক trade-off বেছে নিতে দেয়।

**১০. Idempotency এবং consistency প্রায়ই একসাথে আলোচিত হলেও এই দুটি কেন একই জিনিস নয়?**
Consistency model বর্ণনা করে অন্য operation-এর সাপেক্ষে একটি read কী data observe করার অনুমতিপ্রাপ্ত। Idempotency বর্ণনা করে একটি operation পুনরাবৃত্তি করলে ফলাফল পরিবর্তন হয় কিনা। একটি system strongly consistent হয়েও non-idempotent operation থাকতে পারে (যেমন, একটি linearizable "increment counter" operation), এবং একটি system eventually consistent হয়েও প্রতিটি operation idempotent হতে পারে (যেমন, idempotent "set" operation যার শুধু last-write-wins দরকার)। এরা পারস্পরিক ক্রিয়া করে — দুর্বল consistency-এর অধীনে retry-এর অতিরিক্ত যত্ন দরকার — কিন্তু তারা ভিন্ন ভিন্ন প্রশ্নের উত্তর দেয়।

**১১. Session consistency (read-your-writes + monotonic reads) ব্যবহারকারী একটি system-এ, একই মুহূর্তে দুইজন ভিন্ন user বৈধভাবে কী দেখতে পারে?**
দুইজন ভিন্ন user global state-এর ভিন্ন ভিন্ন snapshot দেখতে পারে — user A এমন একটি comment দেখতে পারে যা user B এখনও দেখেনি, কারণ session consistency শুধু guarantee করে যে একজন client-এর নিজের view সময়ের সাথে সুসংগত, এই নয় যে সব client তাৎক্ষণিকভাবে converge করে। অনেক consumer app-এর জন্য এটি গ্রহণযোগ্য কারণ প্রতিটি user শুধু তার নিজের interaction history-তে অসঙ্গতি লক্ষ্য করে, অন্য user-দের view-এর সাথে তুলনায় নয়।

**১২. একাউন্টের মধ্যে টাকা স্থানান্তরকারী একটি বিদ্যমান non-idempotent "POST /transfer" endpoint-এ আপনি কীভাবে idempotency retrofit করবেন?**
একটি required `Idempotency-Key` (বা `transfer_id`) parameter যোগ করুন যা প্রতিটি transfer attempt-এ client-side-এ তৈরি হয়। Transfers table-এ সেই key-এর উপর একটি unique constraint যোগ করুন, এবং একটি single atomic transaction-এ transfer এবং key-insertion উভয়ই process করুন। একই key দিয়ে যেকোনো retry-তে, insert uniqueness check fail করে (বা একটি lookup বিদ্যমান row খুঁজে পায়), এবং server টাকা আবার না সরিয়ে পূর্বে record করা result ফেরত দেয়।
