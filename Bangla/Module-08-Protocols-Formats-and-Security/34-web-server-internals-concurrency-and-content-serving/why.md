# কেন এই বিষয়টি গুরুত্বপূর্ণ: Web Server Internals — Concurrency, Threading & Content Serving

> **এক বাক্যে:** একই hardware-এ থাকা দুটি server কতগুলো concurrent connection হ্যান্ডেল করতে পারে তাতে শতগুণ পার্থক্য থাকতে পারে, আর এর কারণ code-এর মান নয় — এটি concurrency model, যা বেশিরভাগ developer কখনো সচেতনভাবে বেছে নেন না।

## এই ধারণার আগের পৃথিবী

একটি server request গ্রহণ করে। প্রতিটির জন্যই handling দরকার। স্পষ্ট model হলো: connection প্রতি একটি thread। এটি সহজ, বোঝা সহজ, এবং প্রতিটি request অন্য কাউকে প্রভাবিত না করেই I/O-এর উপর block করতে পারে।

তারপর 10,000 জন ব্যবহারকারী একসাথে connect করে। দশ হাজার thread, প্রতিটিতে প্রায় 1-8 MB stack, মানে গিগাবাইট গিগাবাইট memory কিছুই না করেই বসে আছে। Operating system তার বেশিরভাগ CPU খরচ করে এমন thread-এর মধ্যে context-switch করতে, যেগুলো এমনিতেই network I/O-এর উপর block হয়ে আছে। Throughput ভেঙে পড়ে — কাজটা কঠিন বলে নয়, বরং model workload-এর সাথে খাপ খায় না বলে। এটাই C10K problem, আর এই কারণেই Nginx-এর অস্তিত্ব।

উল্টো দিকের ব্যর্থতাও ঠিক ততটাই সাধারণ: একটি team একটি async event-loop server গ্রহণ করে কারণ এটি "দ্রুত," তারপর একটি request handler-এর ভেতরে একটি CPU-heavy operation চালায়। সেই একটি blocking call পুরো event loop-কে জমিয়ে দেয়, এবং *প্রতিটি* concurrent request আটকে যায়। যে model সুন্দরভাবে scale করেছিল, সেটাই এখন একটি খারাপ handler-এর কারণে ভয়াবহভাবে ব্যর্থ হয়।

## এটি যেসব সমস্যার সমাধান করে

### ১. উচ্চ connection সংখ্যায় memory এবং context-switch overhead
**যা আপনি দেখবেন:** একটি server যা 500টি concurrent connection ঠিকভাবে হ্যান্ডেল করে কিন্তু 5,000-এ গিয়ে ভেঙে পড়ে, memory শেষ হয়ে যায় এবং CPU scheduling-এ পুড়ে যায়।

**কেন এমনটা হয়:** Thread-per-connection প্রতিটি connection-এর জন্য OS-স্তরের resource বরাদ্দ করে, সেই connection কিছু করছে কিনা তা নির্বিশেষে। বেশিরভাগ connection, বেশিরভাগ সময়ে, idle থাকে — network-এর জন্য অপেক্ষা করে।

**Event-driven server গুলো কীভাবে এটি সমাধান করে:** একটি একক thread (বা একটি ছোট pool, প্রতি core-এ একটি) `epoll`/`kqueue` ব্যবহার করে হাজার হাজার socket পর্যবেক্ষণ করে এবং শুধু যেগুলো প্রস্তুত সেগুলোর জন্যই কাজ করে। Connection প্রতি memory কমে kilobyte-এ নেমে আসে। এই কারণেই Nginx এমন hardware-এ লক্ষ লক্ষ connection serve করে যেখানে Apache-এর prefork model মারা যেত — এবং এই কারণেই এটি WebSockets এবং SSE-এর মতো দীর্ঘস্থায়ী connection-এর জন্য অত্যন্ত গুরুত্বপূর্ণ।

### ২. একটি ধীর handler সবকিছু জমিয়ে দেওয়া
**যা আপনি দেখবেন:** একটি async server, যেখানে image resizing বা একটি synchronous database call করা একটি একক endpoint পুরো process-কে অপ্রতিক্রিয়াশীল করে তোলে।

**কেন এমনটা হয়:** একটি event loop cooperative। যেকোনো handler যা yield করে না, সেই loop-এর অন্য প্রতিটি চলমান request-কে block করে দেয়।

**Model বোঝা কীভাবে এটি সমাধান করে:** এটি সেই নিয়ম বলে দেয় যা async কে কাজ করায়: loop কখনো block করবেন না। CPU-bound কাজ একটি worker pool বা আলাদা service-এ যাবে; প্রতিটি I/O call অবশ্যই non-blocking variant হতে হবে। এটি কোনো performance tip নয় — এটি model-এর operating constraint, আর এটি লঙ্ঘন করলে আপনার সবচেয়ে ভালো পরিস্থিতিই আপনার সবচেয়ে খারাপ পরিস্থিতিতে পরিণত হয়।

### ৩. Application server গুলো এমন কাজ করছে যাতে তারা ভয়ানক
**যা আপনি দেখবেন:** আপনার Python বা Node process তার বেশিরভাগ সময় disk থেকে image file পড়ে এবং সেগুলো stream করে বের করে দিতে ব্যয় করছে, এদিকে প্রকৃত request গুলো তাদের পেছনে সারি বেঁধে অপেক্ষা করছে।

**কেন এমনটা হয়:** একটি application runtime-এর মাধ্যমে static file serve করার মানে হলো প্রতিটি byte application-এর memory এবং interpreter-এর মধ্য দিয়ে যাওয়া।

**এই বিষয়টি কীভাবে এটি সমাধান করে:** এটি দেখায় কেন standard architecture হলো একটি application server-এর সামনে একটি reverse proxy (Nginx)। Nginx kernel-level `sendfile` ব্যবহার করে সরাসরি disk থেকে static content serve করে, কখনো data-কে user space-এ copy না করেই, এবং TLS termination, compression, এবং ধীর client গুলো হ্যান্ডেল করে — এদিকে application server শুধুমাত্র দ্রুত, সম্পূর্ণ, proxy করা request-ই দেখে। কাজের এই বিভাজন বেশিরভাগ application-স্তরের optimization-এর চেয়ে বেশি মূল্যবান।

### ৪. ধীর client গুলো আপনার capacity জিম্মি করে রাখা
**যা আপনি দেখবেন:** ভয়াবহ connection-এ থাকা কিছু client, অথবা একটি ইচ্ছাকৃত slowloris attack, request গুলো একবারে এক byte করে ট্রিকল করে আপনার সমস্ত application worker গ্রাস করে ফেলে।

**কেন এমনটা হয়:** যদি application server সরাসরি client-এর সাথে কথা বলে, তাহলে একটি ধীর transfer-এর পুরো সময় জুড়ে একটি worker আটকে থাকে।

**Proxy-তে buffering কীভাবে এটি সমাধান করে:** Nginx ধীর read এবং ধীর write শুষে নেয়, application-কে শুধুমাত্র সম্পূর্ণভাবে buffer করা request হস্তান্তর করে এবং response সাথে সাথেই গ্রহণ করে। আপনার ব্যয়বহুল application worker গুলো শুধু প্রকৃত কাজেই ব্যস্ত থাকে।

## যে মূল্য আপনাকে দিতে হয়

- **Async code লেখা এবং debug করা কঠিন।** Callback এবং promise chain, async-context propagation, এবং তাদের ইতিহাস হারানো stack trace — এগুলো বাস্তব খরচ। একটি সূক্ষ্ম blocking call সহজে ঢুকে পড়ে এবং খুঁজে বের করা কঠিন।
- **Thread-per-connection সেকেলে হয়ে যায়নি।** মাঝারি connection সংখ্যা সহ CPU-bound workload-এর জন্য, thread সহজ এবং স্বাভাবিকভাবেই একাধিক core ব্যবহার করে। আর virtual/green thread (Go-র goroutine, Java 21-এর virtual thread) পুরনো trade-off-কে অনেকটাই দূর করে দেয়, thread-style code দিয়ে event-loop-style scaling দিয়ে — এটা জানা জরুরি, কারণ এটি নতুন system-এর জন্য উত্তর পাল্টে দেয়।
- **Event loop-এর জন্য স্পষ্ট multi-core handling দরকার।** একটি loop একটি core ব্যবহার করে। আপনার প্রয়োজন প্রতি core-এ একটি process এবং একটি load-balancing কৌশল, যা অতিরিক্ত configuration এবং in-process state-কে জটিল করে তোলে।
- **আরও নড়াচড়া করা অংশ।** একটি reverse proxy tier যোগ করা মানে configure, monitor করার এবং ভুল করার জন্য আরও একটি component।
- **Tuning বাস্তব কাজ।** File descriptor limit, backlog size, keep-alive timeout, এবং worker সংখ্যা — এই সবগুলোর default মান উচ্চ-concurrency system-এর জন্য ভুল।

## কখন এটি প্রয়োজন — এবং কখন নয়

| যখন Event-driven / async | যখন Thread-based |
|---|---|
| অনেক concurrent, বেশিরভাগই idle connection | Request গুলো CPU-bound |
| দীর্ঘস্থায়ী connection (WebSockets, SSE, streaming) | Connection সংখ্যা মাঝারি |
| Workload I/O-bound (সাধারণ ক্ষেত্র) | Team-এর পরিচিতি এবং সরলতা বেশি গুরুত্বপূর্ণ |
| আপনি static content serve করছেন বা proxy করছেন | আপনার runtime virtual thread প্রদান করে |

## কেন এটি Interview-এ আসে

এটি একটি গভীরতা-যাচাইকারী প্রশ্ন: "একটি server কতগুলো concurrent connection হ্যান্ডেল করতে পারে?" অথবা "যখন 100,000 ব্যবহারকারী connect করে তখন কী হয়?" দুর্বল উত্তর হলো যুক্তি ছাড়া একটি সংখ্যা। শক্তিশালী উত্তর ব্যাখ্যা করে যে এটি concurrency model-এর উপর নির্ভর করে, connection প্রতি memory অনুমান করে, উল্লেখ করে যে idle connection একটি event loop-এ সস্তা কিন্তু thread-এ ব্যয়বহুল, এবং TLS, static content, এবং ধীর client হ্যান্ডেল করার জন্য সামনে Nginx বসানোর কথা উল্লেখ করে। Chat এবং streaming design-এ, প্রতি-server connection capacity সংখ্যা সরাসরি নির্ধারণ করে আপনার diagram-এ কতগুলো server দেখা যাবে — তাই এটি trivia নয়, এটি capacity planning।

## এটি কীভাবে সংযুক্ত

এটি **HTTP** (topic 6) এবং **transport protocol** (topic 33)-এর নিচে থাকা server-side implementation layer। এটি নির্ধারণ করে scale-এ **WebSockets এবং SSE** (topic 10) কতটা সম্ভব। Reverse proxy tier হলো **topic 8**, static content serving হস্তান্তর করা হয় **CDN**-এ (topic 18), এবং প্রতি-instance capacity হলো **horizontal scaling** (topic 4) এবং **load balancing** (topic 7) সিদ্ধান্তের ইনপুট। Deploy-এর সময় connection draining সরাসরি যুক্ত **zero-downtime deployment**-এর (topic 45) সাথে।

**পরবর্তী:** [Message Formats: JSON, XML & Protocol Buffers](../35-message-formats-json-xml-and-protocol-buffers/why.md) — সেই connection গুলো আসলে কী বহন করে।
