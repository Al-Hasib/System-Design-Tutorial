# অনুশীলন ও Interview প্রশ্ন

**১. Batch processing এবং stream processing-এর মধ্যে মৌলিক পার্থক্য কী?**
Batch processing একটা নির্দিষ্ট সময় ধরে সংগৃহীত bounded, finite dataset-এর উপর কাজ করে এবং সবকিছু একসাথে একটা discrete job হিসেবে প্রসেস করে। Stream processing একটা unbounded, ক্রমাগত ডেটার প্রবাহের উপর কাজ করে, প্রতিটা event (অথবা micro-batch) যেভাবে আসে সেভাবে প্রসেস করে, dataset-এর কোনো সংজ্ঞায়িত শেষ ছাড়াই।

**২. Stream processing-এর তুলনায় batch processing সাধারণত কেন বেশি throughput efficiency অর্জন করে?**
কারণ batch processing পুরো dataset একসাথে নিয়ে কাজ করে, এটা bulk optimization প্রয়োগ করতে পারে — sorting, efficient parallel algorithm, এবং একটা বড়, জানা পরিমাণ ডেটার উপর compute resource-এর পূর্ণ ব্যবহার — বরং streaming-এর মতো প্রতিটা item আলাদাভাবে এবং ক্রমাগত handle করার per-event overhead বহন না করে।

**৩. Stream processing-এ windowing কী, এবং এটা কেন প্রয়োজনীয়?**
Windowing হলো একটা unbounded stream-কে finite chunk-এ (windows) ভাগ করার কৌশল যাতে counts, sums, বা averages-এর মতো aggregate গণনা করা যায়। এটা প্রয়োজনীয় কারণ একটা stream-এর কোনো স্বাভাবিক শেষ নেই, তাই windowing ছাড়া "একটা average" বা "একটা count" ঠিক কী নিয়ে গণনা করা হচ্ছে তা সংজ্ঞায়িত করার কোনো উপায় নেই।

**৪. Tumbling window এবং sliding window-এর মধ্যে পার্থক্য ব্যাখ্যা করুন।**
একটা tumbling window stream-কে fixed-size, non-overlapping, পরপর interval-এ ভাগ করে (যেমন, প্রতিটা আলাদা ৬০-সেকেন্ড block)। একটা sliding window-ও fixed-size কিন্তু পার্শ্ববর্তী windows-এর সাথে overlap করে, এর size-এর চেয়ে ছোট interval-এ পুনরায় গণনা করা হয় (যেমন, একটা ৬০-সেকেন্ড window প্রতি ১০ সেকেন্ডে পুনরায় গণনা করা হয়), তাই একই events একাধিক windows-এ অবদান রাখতে পারে।

**৫. Event time এবং processing time-এর মধ্যে পার্থক্য কী, এবং এটা কেন গুরুত্বপূর্ণ?**
Event time হলো বাস্তব জগতে কিছু আসলে কখন ঘটেছে; processing time হলো system আসলে কখন সেই event handle করেছে। এগুলো গুরুত্বপূর্ণ কারণ network delay, retry, এবং out-of-order delivery এদের যথেষ্ট আলাদা করে দিতে পারে, এবং সময়ের ভুল ধারণার ভিত্তিতে windowed aggregate গণনা করলে ভুল বা বিভ্রান্তিকর ফলাফল হতে পারে (যেমন, একটা purchase-কে ভুল মিনিটে দায়ী করা)।

**৬. Stream processing-এ watermark কী?**
Watermark হলো একটা heuristic marker যা stream processor ব্যবহার করে অনুমান করে (event time-এ) সে এখনো কতটা পিছিয়ে থাকা late-arriving data পেতে পারে, যা system-কে সিদ্ধান্ত নিতে দেয় কিছু ডেটা এখনো দেরিতে আসার সম্ভাবনা সত্ত্বেও একটা time window-এর ফলাফল চূড়ান্ত করে emit করা কখন নিরাপদ।

**৭. পরিস্থিতি: আপনাকে লক্ষ লক্ষ transaction aggregate করে একটা সাপ্তাহিক sales report তৈরি করতে হবে, এবং পরদিন সকালে report প্রস্তুত হলেই চলবে। আপনি batch নাকি stream processing ব্যবহার করবেন, এবং কেন?**
এখানে batch processing বেশি উপযুক্ত কারণ freshness requirement শিথিল (পরদিন সকাল গ্রহণযোগ্য) এবং workload একটা বড়, bounded dataset-এর উপর bulk optimization থেকে উপকৃত হয়, যা sub-second ফলাফলের প্রয়োজন নেই এমন কিছুর জন্য ক্রমাগত stream processing চালানোর চেয়ে এটাকে বেশি cost- এবং resource-efficient করে তোলে।

**৮. পরিস্থিতি: একটা fraudulent credit card transaction approve হওয়ার আগেই আপনাকে detect ও block করতে হবে। আপনি batch নাকি stream processing ব্যবহার করবেন, এবং কেন?**
এখানে stream processing প্রয়োজন কারণ সিদ্ধান্তটা transaction-এর সংক্ষিপ্ত authorization window-এর মধ্যেই (মিলিসেকেন্ড থেকে কয়েক সেকেন্ড) নিতে হবে — একটা রাতের batch job-এর জন্য অপেক্ষা করা মানে detect হওয়ার সময় ততক্ষণে fraudulent transaction-টা ইতিমধ্যেই সম্পন্ন হয়ে গেছে।

**৯. Lambda architecture কী, এবং এটা কোন সমস্যা সমাধানের চেষ্টা করে?**
Lambda architecture আসা ডেটাকে দুটো parallel path-এর মধ্য দিয়ে চালায়: একটা batch layer যা পর্যায়ক্রমে accurate, comprehensive ফলাফল গণনা করে, এবং একটা speed layer যা দ্রুত, আনুমানিক real-time ফলাফল গণনা করে; একটা serving layer query-র জন্য দুটো view merge করে। এটা একই সাথে batch processing-এর accuracy/completeness এবং stream processing-এর low latency চাওয়ার সমস্যা সমাধান করে।

**১০. Lambda architecture-এর প্রধান দুর্বলতা কী, এবং Kappa architecture কীভাবে এটা মোকাবিলা করে?**
Lambda-র প্রধান দুর্বলতা হলো আপনাকে একই রকম business logic implement করা দুটো আলাদা codebase (একটা batch-এর জন্য, একটা streaming-এর জন্য) তৈরি, maintain, এবং সমন্বয়ে রাখতে হয়, যা একটা operational এবং engineering burden। Kappa architecture এটা মোকাবিলা করে সব ডেটাকে — historical data সহ — একটা একক replayable stream হিসেবে ধরে, একটা codebase ব্যবহার করে, এবং পুনর্গণনা প্রয়োজন হলে stream-টাকে কেবল শুরু থেকে reprocess করে।

**১১. Kappa architecture-কে ব্যবহারিক করে তোলে কী, এবং এটা কোন underlying technology-র উপর নির্ভর করে?**
Kappa architecture ব্যবহারিক কারণ Apache Kafka-র মতো durable, replayable log-based systems আছে, যা historical events ধরে রাখে এবং consumers-কে তাদের offset reset করে যেকোনো সময় থেকে ডেটা reprocess করতে দেয় — এমন একটা system ছাড়া, একটা stream-only pipeline-এর মধ্য দিয়ে history "replay" করার কোনো উপায় থাকত না।

**১২. আলাদা features-এর জন্য batch এবং stream processing দুটোই ব্যবহার করে এমন একটা single product-এর বাস্তব উদাহরণ দিন, এবং কেন তা ব্যাখ্যা করুন।**
একটা music streaming service সাপ্তাহিক একটা personalized playlist তৈরি করতে batch processing ব্যবহার করতে পারে (যেমন, "Discover Weekly"), কারণ এক মাসের listening history-র উপর সপ্তাহে একবার ব্যয়বহুল machine learning model চালানো efficient এবং সেই feature-এর জন্য staleness গ্রহণযোগ্য। একই service একটা live "currently listening" counter অথবা trending-song detection চালাতে stream processing ব্যবহার করতে পারে, কারণ সেগুলোতে সপ্তাহে একবার নয়, সেকেন্ডের মধ্যে activity প্রতিফলিত হতে হবে।
