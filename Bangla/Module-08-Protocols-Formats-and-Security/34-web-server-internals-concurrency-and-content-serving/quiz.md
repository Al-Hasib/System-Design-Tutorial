# অনুশীলন ও Interview প্রশ্নসমূহ (Practice & Interview Questions)

**১. প্রতিটি request-এর জন্য প্রতিটি web server যে চারটি ধাপ সম্পাদন করে সেগুলো কী কী?**
Connection accept করা, request পড়া এবং parse করা, তা process করা (response তৈরির জন্য যা প্রয়োজন), এবং response ফেরত লেখা।

**২. C10K problem কী, এবং কেন এটি web server design-এ পরিবর্তন আনতে বাধ্য করেছিল?**
এটি একটি single server-এ 10,000+ concurrent connection হ্যান্ডেল করার ঐতিহাসিক অসুবিধা। Thread-per-connection এবং process-per-connection model গুলো এতদূর scale করতে connection প্রতি অনেক বেশি memory এবং OS scheduling overhead খরচ করে, যা industry-কে event-loop, non-blocking I/O architecture-এর দিকে ঠেলে দিয়েছিল, যেগুলো প্রতিটি idle, অপেক্ষারত connection-এর জন্য একটি thread উৎসর্গ করে না।

**৩. Blocking এবং non-blocking I/O-এর মধ্যে পার্থক্য ব্যাখ্যা করুন।**
Blocking I/O operation (যেমন একটি DB query) সম্পন্ন না হওয়া পর্যন্ত calling thread-কে সম্পূর্ণভাবে থামিয়ে দেয়। Non-blocking I/O operation জারি করে এবং thread-কে অন্য কাজ চালিয়ে যেতে দেয়, শুধুমাত্র operation শেষ হলেই (একটি callback বা event-এর মাধ্যমে) notify করা হয়।

**৪. কেন একটি একক CPU-bound task "event loop block" করতে পারে, এবং কেন এটি বিপজ্জনক?**
একটি event loop সাধারণত একটি (বা কয়েকটি) thread-এ চলে, যাদের অন্য প্রতিটি connection-এর জন্যও I/O callback পাঠাতে হয়। যদি একটি কাজ yield না করে দীর্ঘ synchronous computation চালায়, তাহলে সেই thread শেষ না হওয়া পর্যন্ত অন্য কোনো connection সার্ভ করতে পারে না — শুধু ধীর request-টিই নয়, loop যে সমস্ত request হ্যান্ডেল করছিল সবগুলোই আটকে যায়।

**৫. Image এবং CSS file-এর মতো static asset কেন সরাসরি আপনার application server-এর request-handling code দিয়ে serve করা উচিত নয়?**
এগুলোর প্রতি-request কোনো computation প্রয়োজন হয় না — প্রতিবারই একই byte — তাই এগুলোকে application-এর concurrency model-এর মধ্য দিয়ে route করা সেই capacity নষ্ট করে যা dynamic, request-নির্দিষ্ট কাজের জন্য সংরক্ষিত থাকা উচিত। এগুলোর জায়গা একটি CDN বা reverse-proxy edge-এ, যা এগুলোকে অনেক কম খরচে এবং ব্যবহারকারীর কাছাকাছি থেকে serve (এবং cache) করতে পারে।

**৬. একটি ধীর database query কীভাবে হ্যান্ডেল করে তার ভিত্তিতে thread-per-request এবং event-loop model তুলনা করুন।**
Thread-per-request: dedicated thread block হয়ে যায় এবং query ফেরত না আসা পর্যন্ত অন্য কিছুই করে না, পুরো সময় জুড়ে সেই thread-এর memory এবং scheduling slot আটকে রাখে। Event loop: query non-blocking-ভাবে জারি করা হয়, loop অন্য connection সার্ভ করতে এগিয়ে যায়, এবং query-র result এলেই শুধু একটি callback এই request পুনরায় শুরু করে — কোনো thread অলস বসে অপেক্ষা করে না।

**৭. একটি event-loop-ভিত্তিক server-এ, অন্য request-এর ক্ষতি না করে সত্যিকারের CPU-heavy কাজ (যেমন, PDF generation, image resizing) হ্যান্ডেল করার একটি ব্যবহারিক উপায় কী?**
এটিকে একটি আলাদা worker pool, background job queue, বা dedicated service-এ offload করুন (এটি ঠিক Module 5-এর message-queue/async-processing pattern) যাতে CPU-heavy কাজ event loop thread-এর বাইরে চলে এবং অন্য connection-এর I/O callback-কে block না করে।

**৮. কেন Nginx সাধারণত connection প্রতি একটি thread-এর বদলে একটি ছোট, নির্দিষ্ট সংখ্যক worker process চালায়?**
প্রতিটি Nginx worker process নিজস্ব event loop চালায় যা একইসাথে হাজার হাজার non-blocking connection হ্যান্ডেল করতে সক্ষম, তাই কয়েকটি worker (প্রায়ই CPU core সংখ্যার সাথে মিলিয়ে) একটি one-thread-per-connection model-এর চেয়ে অনেক বেশি সমসাময়িক connection সার্ভ করতে পারে, একইসাথে CPU core জুড়ে সত্যিকারের parallelism-ও করতে দেয়।

**৯. পরিস্থিতি: আপনার Node.js API-র response time load-এর নিচে ভয়াবহভাবে বেড়ে যাচ্ছে, যদিও CPU usage কম দেখাচ্ছে এবং database সুস্থ। আজকের বিষয়ের সাথে যুক্ত সম্ভাব্য কারণ কী?**
Request path-এর কোথাও একটি blocking বা দীর্ঘ-চলমান synchronous operation (যেমন, একটি বিশাল payload-এর synchronous JSON parsing, একটি tight synchronous loop, অথবা একটি blocking library call) event loop-কে জমিয়ে দিচ্ছে, database এবং সামগ্রিক CPU utilization ঠিক দেখালেও অন্য প্রতিটি চলমান request দেরি করে দিচ্ছে।

**১০. সত্য নাকি মিথ্যা: একটি event-loop architecture সম্পূর্ণভাবে যেকোনো thread বা worker process-এর প্রয়োজনীয়তা দূর করে দেয়।**
মিথ্যা। বেশিরভাগ production system hybrid — একটি async event loop I/O-bound কাজ দক্ষতার সাথে হ্যান্ডেল করে, কিন্তু CPU-bound বা blocking কাজ এখনো একটি আলাদা thread pool, worker process pool, বা background job system-এ offload করা হয় যাতে এটি event loop-কে জমিয়ে না দেয়।
