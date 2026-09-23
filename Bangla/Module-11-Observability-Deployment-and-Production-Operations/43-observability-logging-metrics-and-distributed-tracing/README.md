# Observability: Logging, Metrics & Distributed Tracing

**কঠিনতার মাত্রা:** মধ্যম (Intermediate)

## শেখার উদ্দেশ্য (Learning Objectives)

- observability মানে কী এবং কেন এটা শুধু "logs থাকা"-র চেয়ে আলাদা, তা ব্যাখ্যা করা।
- observability-এর তিনটি স্তম্ভ — logs, metrics, এবং traces — এবং প্রতিটি কোন প্রশ্নের উত্তর দিতে সবচেয়ে ভালো, তা বর্ণনা করা।
- distributed tracing কীভাবে একটি একক trace ID কে service boundary জুড়ে প্রচার (propagate) করে একটি request-এর সম্পূর্ণ পথ পুনর্গঠন করে, তা ব্যাখ্যা করা।
- monitoring (পরিচিত failure mode গুলো পর্যবেক্ষণ করা) কে observability (অজানা failure mode গুলো তদন্ত করতে পারা) থেকে আলাদা করা।
- একটি basic alerting strategy ডিজাইন করা যা missed incident এবং alert fatigue — দুটোই এড়িয়ে চলে।

## স্ক্রিপ্ট (Script)

### Hook / Intro

এই কোর্সে এ পর্যন্ত আমরা যত সিস্টেম ডিজাইন করেছি, সেগুলো সবই হোয়াইটবোর্ডে বক্স এবং তীর দিয়ে আঁকা হয়েছে। প্রোডাকশনে, সেই কাঠামোর কোনোটাই আপনার কাছে ডিফল্টভাবে দৃশ্যমান নয় — একটি request আসে, আপনার সিস্টেমের ভেতরে হারিয়ে যায়, এবং হয় একটি response বের হয়ে আসে অথবা আসে না, আর যদি কিছু ভুল হয়, তাহলে আপনার কাছে ঠিক ততটুকু তথ্যই থাকবে যতটা আগেই আপনি সুনির্দিষ্টভাবে ক্যাপচার করার সিদ্ধান্ত নিয়েছিলেন — এক বাইটও বেশি নয়। এটাই observability: এমনভাবে সিস্টেম তৈরি করার চর্চা যাতে কিছু ভুল হলে — বিশেষত এমন কিছু যা আপনি কখনো আগে থেকে অনুমান করেননি — আপনি বাইরে থেকে, লাইভ প্রোডাকশন প্রসেসে debugger সংযুক্ত না করেই, আসলে কী ঘটেছিল তা বের করতে পারেন। আজ আমরা এই তিনটি স্তম্ভ কভার করব যা এটাকে সম্ভব করে তোলে, এবং কেন microservices architecture (Module 7 মনে করুন) এটাকে দেখতে যতটা সহজ মনে হয় তার চেয়ে অনেক বেশি কঠিন করে তোলে।

### Logs: এখানে, এখনই কী ঘটেছে

একটি **log** হলো একটি নির্দিষ্ট ঘটনার timestamp-যুক্ত, স্বতন্ত্র রেকর্ড: "user 5 logged in," "payment failed with error X," "cache miss for key Y।" Logs হলো সবচেয়ে সূক্ষ্ম, বিস্তারিত স্তম্ভ — এটি আপনাকে বলতে পারে একটি নির্দিষ্ট প্রসেসের একটি নির্দিষ্ট মুহূর্তে ঠিক কী ঘটেছিল। বড় স্কেলে ব্যবহারিক চ্যালেঞ্জটা log লেখা নয় — প্রতিটি ইঞ্জিনিয়ারই সেটা করে — চ্যালেঞ্জ হলো যখন আপনার কাছে শত শত service instance-এ ছড়িয়ে থাকা লক্ষ লক্ষ log থাকে, তখন সেগুলোকে কাজে লাগানো। এই কারণেই প্রোডাকশন সিস্টেমগুলো সর্বজনীনভাবে **structured logging**-এর দিকে ঝুঁকে পড়ে: একটি ফ্রি-টেক্সট বাক্যের বদলে, প্রতিটি log entry হলো একটি structured record (সাধারণত JSON) যাতে সামঞ্জস্যপূর্ণ field থাকে — timestamp, service name, request ID, severity level, এবং যা যা প্রাসঙ্গিক context — যা একটি কেন্দ্রীয় log aggregation সিস্টেমে (যেমন ELK stack — Elasticsearch, Logstash, Kibana — অথবা একটি managed সমতুল্য) পাঠানো হয়, যেখানে সেগুলো আসলেই খোঁজা, ফিল্টার করা এবং correlate করা যায়, আলাদা আলাদা সার্ভারে ছড়িয়ে থাকা এমন কিছু অনুসন্ধানযোগ্য টেক্সট ফাইল হিসেবে পড়ে থাকার বদলে যেগুলোর জন্য আপনাকে একে একে SSH করতে হতো।

### Metrics: সিস্টেম কেমন করছে, সময়ের সাথে?

একটি **metric** হলো সময়ের সাথে সমষ্টিকৃত একটি সাংখ্যিক পরিমাপ — requests per second, p99 latency, error rate, CPU utilization, queue depth। একটি log যেখানে একটি নির্দিষ্ট ঘটনা সম্পর্কে বলে, সেখানে একটি metric হাজার হাজার বা লক্ষ লক্ষ ঘটনা জুড়ে আচরণের *আকৃতি* সম্পর্কে বলে, স্বল্প খরচে, কারণ এটি প্রতিটি individual data point সংরক্ষণ করার বদলে আগে থেকে aggregate করা থাকে। এটাই dashboard-এর এবং, গুরুত্বপূর্ণভাবে, **alerting**-এর চালিকাশক্তি: আপনি চান না যে একজন মানুষ ২৪/৭ একটি dashboard দেখুক, আপনি চান এমন একটি সিস্টেম যা error rate কোনো threshold পার হলে বা latency সম্মত সীমার বাইরে চলে গেলেই সাথে সাথে কাউকে page করবে। এখানে মানসম্মত toolchain হলো Prometheus-এর মতো কিছু (যা নিয়মিত interval-এ service থেকে metric "scrape" করে এবং সেগুলোকে time series হিসেবে সংরক্ষণ করে) visualization-এর জন্য Grafana-র সাথে জোড়া লাগানো — এবং একটি ভালোভাবে ডিজাইন করা alerting strategy সেসব **symptom**-এর উপর page করে যা আসলেই ব্যবহারকারীদের প্রভাবিত করে (elevated error rate, breached latency SLO) প্রতিটি সম্ভাব্য অভ্যন্তরীণ কারণের উপর নয়, যা সেই খুবই বাস্তব, খুবই সাধারণ failure mode প্রতিরোধ করে যাকে বলে **alert fatigue**: এত বেশি low-signal page যে ইঞ্জিনিয়াররা সবগুলোকে উপেক্ষা করা শুরু করে, এমনকি যেটা গুরুত্বপূর্ণ সেটাকেও।

### Distributed Tracing: এই *নির্দিষ্ট* request-এর সাথে কী ঘটেছিল, প্রতিটি service জুড়ে যা এটি স্পর্শ করেছে?

এখানেই microservices architecture (Module 7) সত্যিকার অর্থে খেলা বদলে দেয়। একটি monolith-এ, একটি ধীর request হলো একটি প্রসেস যা আপনি সরাসরি profile করতে পারেন। একটি microservices architecture-এ, একটি একক user-facing request হয়তো এক ডজন অভ্যন্তরীণ service call-এ ছড়িয়ে যেতে পারে — logs এবং metrics একা, প্রতিটি service থেকে স্বাধীনভাবে সংগৃহীত, আপনাকে বলে না ওই এক ডজন call-এর মধ্যে আসলে কোনটা ধীর ছিল, বা একটি নির্দিষ্ট request কোথায় ব্যর্থ হয়েছিল তার পথে। **Distributed tracing** এটা সমাধান করে একটি unique **trace ID** তৈরি করে যখনই একটি request সিস্টেমে প্রবেশ করে, এবং সেই একই trace ID কে request যে প্রতিটি downstream service call স্পর্শ করে তার মধ্য দিয়ে প্রচার (propagate) করে — সাধারণত একটি HTTP header হিসেবে (Module 8 থেকে gRPC/HTTP মনে করুন) যা প্রতিটি service পড়ে, নিজের কাজের পাশাপাশি log করে, এবং যা কিছু পরে call করে তার কাছে ফরওয়ার্ড করে। ওই trace-এর মধ্যে প্রতিটি স্বতন্ত্র কাজের একক (একটি service-এর request-এর নিজের অংশ পরিচালনা) একটি **span** হিসেবে রেকর্ড করা হয়, যার একটি start time, duration, এবং নিজস্ব metadata থাকে, এবং span গুলো প্রকৃত call graph-এর সাথে মিলিয়ে একটি parent-child গাছে যুক্ত থাকে। Jaeger, Zipkin, বা একটি managed সমতুল্যের মতো টুল তখন আপনাকে একটি নির্দিষ্ট ধীর বা ব্যর্থ request তুলে ধরতে এবং দৃশ্যতভাবে ঠিক দেখতে দেয় সেই এক ডজন service-এর মধ্যে কোনগুলো এটি স্পর্শ করেছে, কোন ক্রমে, এবং ঠিক কোনটা 800ms নিয়েছে যখন বাকিরা প্রত্যেকে 5ms নিয়েছে। এটা "checkout flow মাঝে মাঝে ধীর হয়" — এই রহস্যকে পাঁচ মিনিটের একটি তদন্তে রূপান্তরিত করে।

### Monitoring বনাম Observability

একটি পার্থক্য সম্পর্কে স্পষ্ট থাকা মূল্যবান যা প্রায়শই ঝাপসা হয়ে যায়: **monitoring** হলো একটি পূর্বনির্ধারিত সেট পরিচিত failure mode পর্যবেক্ষণ করা — "CPU 90% পার হলে alert করো," "health check ব্যর্থ হলে alert করো।" এটি সেসব প্রশ্নের উত্তর দেয় যা আপনি আগে থেকেই জিজ্ঞাসা করার কথা ভেবেছিলেন। **Observability** হলো বৃহত্তর একটি বৈশিষ্ট্য — অপ্রত্যাশিত কিছু ঘটার পরে আপনার সিস্টেমের অভ্যন্তরীণ অবস্থা সম্পর্কে *নতুন* প্রশ্ন জিজ্ঞাসা করতে পারার ক্ষমতা, নতুন কোড শিপ না করেই যাতে আপনি এখন বুঝতে পারা সুনির্দিষ্ট instrumentation যোগ করতে পারেন। একটি সিস্টেম ভারীভাবে monitored হতে পারে (কয়েক ডজন dashboard এবং alert) এবং তবুও observable না হতে পারে, যদি ওই dashboard গুলোর কোনোটাই ঘটনাক্রমে সেই নির্দিষ্ট, novel failure mode ক্যাপচার না করে যা মাত্র ঘটেছে। সমৃদ্ধ structured logs, উপযোগী dimension সহ granular metrics (শুধু "error rate" নয় বরং "error rate by endpoint, by region, by client version"), এবং distributed tracing — একসাথে এগুলোই একটি সিস্টেমকে observable করে তোলে — সত্যিকার অর্থে, ঘটনার পরে তদন্তযোগ্য, এমন failure mode-এর জন্য যা কেউ সুনির্দিষ্টভাবে অনুমান করেনি।

### বাস্তব-জগতের উদাহরণ (Real-World Example)

কল্পনা করুন একজন on-call ইঞ্জিনিয়ার page পান: checkout latency-র p99 200ms থেকে বেড়ে 4 সেকেন্ড হয়ে গেছে। Metrics dashboard (Prometheus/Grafana) এই spike নিশ্চিত করে এবং এটিকে নির্দিষ্টভাবে checkout service-এ সীমাবদ্ধ করে, কিন্তু *কেন* তা বলে না। Jaeger-এ প্রভাবিত request গুলোর একটির জন্য একটি slow trace তুলে ধরলে সম্পূর্ণ call graph দেখা যায়: API gateway → checkout service → inventory service → payment service → notification service, প্রতিটি hop-এর সাথে span duration যুক্ত — এবং সাথে সাথেই দেখা যায় যে শুধু payment service call-ই ওই 4 সেকেন্ডের 3.8 সেকেন্ডের জন্য দায়ী, বাকি সবকিছু স্বাভাবিক। এখন তদন্তের পরিধি নির্দিষ্ট: payment service-এর structured logs তুলে ধরুন, ঠিক ওই trace ID দিয়ে ফিল্টার করে, এবং payment processing-এর ভেতরে নির্দিষ্ট error বা ধীর downstream dependency খুঁজে বের করুন — "checkout ধীর" থেকে "payment service-এর এই নির্দিষ্ট downstream call"-এ পৌঁছাতে ঘণ্টার পর ঘণ্টা অনুমান নয়, বরং মিনিটেই।

### সারসংক্ষেপ (Recap)

Observability মানে হলো আপনার সিস্টেমে আসলে কী ঘটেছে তা তদন্ত করতে পারা, যার মধ্যে এমন failure mode-ও রয়েছে যা আপনি কখনো সুনির্দিষ্টভাবে অনুমান করেননি — শুধু একটি পূর্বনির্ধারিত dashboard সেট দেখা নয়। Logs বিস্তারিত, স্বতন্ত্র ঘটনা ক্যাপচার করে; metrics সময়ের সাথে আচরণের সমষ্টিগত আকৃতি ক্যাপচার করে এবং alerting চালায়; distributed tracing একটি request যে প্রতিটি service স্পর্শ করে তার মধ্য দিয়ে একটি trace ID প্রচার করে, যা আপনাকে ঠিক দেখতে দেয় একটি distributed call graph-এর কোন hop আসলে একটি ধীর বা ব্যর্থ request-এর জন্য দায়ী ছিল। Monitoring (পরিচিত failure mode পর্যবেক্ষণ করা) প্রয়োজনীয় কিন্তু যথেষ্ট নয় — প্রকৃত observability-র জন্য তিনটি স্তম্ভই একসাথে কাজ করা দরকার, বিশেষত একবার একটি সিস্টেম একাধিক service-এ বিভক্ত হয়ে গেলে।

### এরপর কী (What's Next)

আমরা কভার করেছি কীভাবে একটি চলমান সিস্টেম আসলে কী করছে তা দেখা যায়। পরবর্তী ভিডিওতে দেখব সেই সিস্টেম আসলে কীভাবে প্যাকেজ করা হয় এবং প্রথমে চালানোর জন্য শিডিউল করা হয় — containers এবং orchestration, সেই infrastructure স্তর যা এই কোর্সে এখন পর্যন্ত আমরা করা প্রতিটি "just deploy it" ধারণার নিচে থাকে।

## মূল বিষয়সমূহ (Key Takeaways)

- Observability মানে হলো ঘটনার পরে অজানা/অপ্রত্যাশিত failure mode তদন্ত করতে পারা, শুধু একটি পূর্বনির্ধারিত dashboard সেট দেখা নয় — সেটা হলো monitoring, যা প্রয়োজনীয় কিন্তু নিজে থেকে যথেষ্ট নয়।
- Logs হলো বিস্তারিত, প্রতি-ঘটনা রেকর্ড; structured logging (JSON, সামঞ্জস্যপূর্ণ field) একটি কেন্দ্রীয় aggregation সিস্টেমে পাঠানো হলো যা সেগুলোকে বড় স্কেলে অনুসন্ধানযোগ্য করে তোলে।
- Metrics হলো সময়ের সাথে আগে থেকে সমষ্টিকৃত সাংখ্যিক পরিমাপ (latency, error rate, throughput) যা dashboard এবং alerting চালায় — alert fatigue এড়াতে user-facing symptom-এর উপর alert করুন।
- Distributed tracing একটি request যে প্রতিটি service স্পর্শ করে তার মধ্য দিয়ে একটি trace ID প্রচার করে, request কে যুক্ত span-এ ভেঙে দেয় যাতে আপনি ঠিক দেখতে পারেন একটি distributed call graph-এর কোন hop ধীরগতি বা ব্যর্থতার কারণ ছিল।
- সমৃদ্ধ structured logs, high-cardinality metrics, এবং distributed tracing — একসাথে, একা কোনো একটি নয় — এটাই একটি distributed সিস্টেমকে সত্যিকার অর্থে observable করে তোলে।
